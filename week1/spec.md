현재 loader.py 는 파일을 가져다 검증 후 /done 또는 /error로 이동
처리 결과를 app.log에 기록 → exporter가 이 로그를 tail하도록 되어 있다, 처리 결과를
누적하는 Counter 방식으로 변환하여 바이너리 헤더 파일의 검증을 누적 모니터링 하고 싶다.
incoming 단계에서만 존재하는 헤더 파일을 매트릭으로 노출하는 것은 의미가 없다. 해당 매트릭은 제거하고
(물론 검증이 루어 져야함)  'magic_ok':       0,   # 헤더 정상 파일 수
'magic_fail':     0,   # 헤더 손상 파일 수
'layout_ok':      0,   # 레이아웃 일치 파일 수
'layout_fail':    0,   # 레이아웃 불일치 파일 수
'version_fail':   0,   # 버전 불일치 파일 수
'total_records':  0,   # 처리한 총 레코드 수

로그 추적 그대로 유지 



헤더 및 데이터레이아웃 검증 재설계

1. file_size - 14 == record_count × 8          (shape 검증)
2. file_size - 14  % 8 == 0                    (잔여 바이트 없음)
3. sum(body_bytes) % 2^32 == checksum % 2^32   (integrity 검증)
4. struct.unpack(f'>{record_count}q', body)    (역직렬화 가능성)

현재 데이터가 최종적으로 저장되는 파티션이 done/error 구분없이 전부 저장되고 있는데 당연히 done으로 완료된 후 done에 해당하는 파일들만 저장해야 한다.