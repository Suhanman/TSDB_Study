## generator.py : 파일생성

MAGIC : DAT1
VERSION : 2

[ 헤더 영역 - 14바이트 ]

00-03: 44 41 54 31         (Magic: "DAT1")
04-05: 00 02               (Version: 2)
06-09: 00 00 00 02         (Count: 2)
0A-0D: XX XX XX XX         (Checksum: 바디 합계)


### counter 와 gauge
Counter는 값이 0부터 시작해서 계속 늘어나기만 하는 누적 메트릭
Gauge는 현재의 상태를 나타내는 수치

