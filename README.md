## repository url
이 repository는 하기 프로젝트에서 fork된 repository이다.  
https://github.com/cantools/cantools
  
개인적으로 프로젝트를 진행하면서 해결하기 어려운 부분을 참고하기 위해  
코드 분석을 해봤다. (CAN DATA를 decoding하는 부분에 대해)  
  
## 설명서
 - rstviewer로 볼 수 있다.  
pip install rstviewer  
rstviewer.exe .\README.rst 
 - 또는 github페이지에서 보면 된다.  

## 패키지 설치
pip install cantools  
pip install uptime  
  
## 실행
python.exe -s -m cantools monitor .\tests\files\dbc\motohawk.dbc  
  
## CAN BUS, Channel 설정 변경  
현재 코드 상으로는 vcan을 사용하고 있는데, 
나는 PEAK-CAN USB를 사용 중이므로 bus타입을 pcan0으로, channel을 PCAN_USBBUS1로 설정해주면 된다.  
  
## DBC 파일 수정  
tests\files\dbc\motohawk.dbc을 열어서 아래와 같이 수정  
BU_ 부분에 my_node1 노드를 추가해주고,  
BO_ 부분에 MY_MESSAGE_1메시지를 추가한 것과 같이 필요한 메시지를 추가한다.  
~~~
BU_: PCM1 FOO my_node1


BO_ 496 ExampleMessage: 8 PCM1
 SG_ Temperature : 0|12@0- (0.01,250) [229.52|270.47] "degK"  PCM1,FOO
 SG_ AverageRadius : 6|6@0+ (0.1,0) [0|5] "m" Vector__XXX
 SG_ Enable : 7|1@0+ (1,0) [0|0] "-" Vector__XXX

BO_ 771 MY_MESSAGE_1: 8 my_node1
    SG_ my_signal1: 0|10@1+ (0.1,0) [0|1] "" my_node2
    SG_ xxx: ...
~~~

## 실행 결과
~~~
Received: 622, Discarded: 0, Errors: 0                                                                                                                
   TIMESTAMP  MESSAGE                                                                                                                                    
	24.551  MY_MESSAGE_1(
				my_signal1: 9.8,
        ...중략...
			)
~~~
  
  
## 간단한 코드 분석  
나의 경우, 아직 개발 완료된 프로젝트가 아니어서 프로토콜이 업데이트되고 있었고, 시뮬레이션 테스트로 검증 작업 중이었다.  
그런데, 기존 코드가 부동소수점 데이터를 수신하지 못하는 문제가 있었다.  
원인을 분석하기 위해 현재 코드와 다른 오픈소스코드와 결과를 비교해보고  
CAN message를 data로 decoding하는 로직을 확인해보기 위해 코드 분석을 해봤다.  
전체를 분석하기에는 시간이 부족해서, 이것 만이라도 다른 필요한 사람들에게 도움이 됬으면 하는 바람에서 기록으로 남긴다.  
~~~
[monitor]              [__utils__]          [message]
  tick()                    |                   |
    |                       |                   |
    |---------|             |                   |
    |<--------|             |                   |
    | update()              |                   |
    |---------|             |                   |
    |<--------|             |                   |
    | update_messages()     |                   |
    |---------|             |                   |
    |<--------|             |                   |
    | try_update_messages() |                   |
    |                       |                   |
    ...                     |                   |
    |---------------------->|                   |
    |       format_message(message, data, True, False)
    |                       |------------------>|
    |                       |message.decode_simple(data, decode_choices, allow_truncated=allow_truncated).
    |                       |                   | the argumant(scaling) is assigned to default value(True).
    |                       |                   |
    |                       |                   |--|
    |                       |                   |  |
    |                       |                   |<-|
    |                       |                   |_decode(_codecs, data, decode_choices, scaling, allow_truncated)
    |                       |                   |
    |                       |                   |--|
    |                       |                   |  |
    |                       |                   |<-|
    |                       |                   |## decide_data()에 전달되는 node dictionary는 Codecs를 참조하는 dictionary임.
    |                       |                   |## Codecs에 대한 데이터 형식은 아래의 별도 설명을 참고.
    |                       |                   |## fields <- node['signals']: fields에는 signal(key)에 해당하는 리스트(value)가 대입됨
    |                       |                   |## formats <- node['formats']: fields에는 formats(key)에 해당하는 리스트(value)가 대입됨
    |                       |                   |decoded = decode_data(data,# XXX: 수신된 message의 byte array부분. 실제 데이터
    |                       |                   |                      self.length,
    |                       |                   |                      node['signals'],# XXX: signal()들을 원소로 가진 리스트
    |                       |                   |                      node['formats'],# XXX: Formats()
    |                       |                   |                      decode_choices,
    |                       |                   |                      scaling,# True..
    |                       |                   |                      allow_truncated)
    |                       |                   | in decide_data(), decode data, please refer to below code.
~~~
~~~python
def decode_data(data: bytes,# XXX: 수신된 message의 byte array부분. 실제 데이터
                expected_length: int,
                fields: Sequence[Union["Signal", "Data"]], # XXX: signal()들을 원소로 가진 리스트
                formats: Formats, # XXX: Formats()
                decode_choices: bool,
                scaling: bool,
                allow_truncated: bool,
                ) -> SignalDictType:

    actual_length = len(data)
    if allow_truncated and actual_length < expected_length:
        data = data.ljust(expected_length, b"\xFF")

    unpacked = {
        **formats.big_endian.unpack(bytes(data)),
        **formats.little_endian.unpack(bytes(data[::-1])),
    }

    if allow_truncated and actual_length < expected_length:
        # remove fields that are outside available data bytes
        valid_bit_count = actual_length * 8
        for field in fields:
            if field.byte_order == "little_endian":
                sequential_startbit = field.start
            else:
                # Calculate startbit with inverted indices.
                # Function body of ``sawtooth_to_network_bitnum()``
                # is inlined for improved performance.
                sequential_startbit = (8 * (field.start // 8)) + (7 - (field.start % 8))

            if sequential_startbit + field.length > valid_bit_count:
                del unpacked[field.name]

    decoded = {}
    for field in fields:
        try:
            value = unpacked[field.name]
            if (field.name == 'my_signal1'):# my custom CAN signal name in message
                print('unpacked: {0}'.format(unpacked))
                print('field.name: {0}, value: {1}'.format(field.name, value))
                print('decode_choices: {0}'.format(decode_choices))

            if decode_choices:
                try:
                    decoded[field.name] = field.choices[value]  # type: ignore[index]
                    continue
                except (KeyError, TypeError):
                    pass

            if scaling:
                if (field.name == 'my_signal1'):
                    print('77777 field scale: {0}, value: {1}, offset: {2}'.format(field.scale, value, field.offset))
                    # field scale: 0.1, value: 231, offset: 0
                # 여기서 부동소수점 처리가 된다.
                # 내가 simulation tool에서 보낸 데이터는 23.1이었다.
                # decoded된 data에는 scale과 value를 곱한 값이 대입된다.(scale * value + offset == 0.1 * 231 + 0)
                decoded[field.name] = field.scale * value + field.offset
                if (field.name == 'my_signal1'):
                    print('77777 decoded value: {0}'.format(decoded[field.name]))
                continue
            else:
                decoded[field.name] = value

        except KeyError:
            if not allow_truncated:
                raise
        
    print('decoded: {0}'.format(decoded))
    return decoded
~~~
