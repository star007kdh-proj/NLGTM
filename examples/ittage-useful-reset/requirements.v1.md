# ITTAGE 요구사항
- source: mas/ittage.md @ ff68f3c5a
- review: pending

- [R-ITTAGE-1] 히트로 예측을 낼 때 출력 target은 그 엔트리를 마지막으로 학습시킨 실제 target과 반드시 같아야 한다. 학습 이후 region 저장소가 교체되거나 가득 찼다는 이유만으로 히트 상태에서 다른 주소를 내보내서는 절대 안 된다.
  - evidence: src/main/scala/xiangshan/frontend/bpu/ittage/Ittage.scala:231-239 | src/main/scala/xiangshan/frontend/bpu/ittage/RegionWays.scala:82-87 | docs/ittage_algorithm.md:369 "rTable에는 ITTAGE 엔트리들이 어떤 pointer를 참조 중인지 역인덱스가 없다. 교체 시 reference invalidation이 없으므로 stale은 영원히 silently 남음"
  - observe: ittage_hit
  - status: proposed

- [R-ITTAGE-2] taken indirect 분기(return 제외)가 resolve될 때마다 예측을 제공했던 엔트리는 반드시 그 결과로 학습되어야 하며, 학습은 예측 당시 조회된 바로 그 엔트리(같은 pc·같은 history)에만 적용되어야 한다.
  - evidence: src/main/scala/xiangshan/frontend/bpu/ittage/Ittage.scala:156-167, 348-360 | commit 7fba9e4b9 "fix(ittage): delay startVAddr & phr from io.train for update (#5244)" | commit f79d7a4f6 "fix(ittage): add training of Call instructions to ittage (#5311)"
  - status: proposed

- [R-ITTAGE-3] provider 대신 alt 예측을 사용했다가 틀렸을 때 alt 엔트리는 반드시 약화되어야 하고, alt와 다른 정답을 낸 provider 엔트리는 반드시 useful로 보호되어야 한다. 오예측이 아닌 경우에는 alt 엔트리를 약화시키거나 새 엔트리를 만들어서는 절대 안 된다.
  - evidence: src/main/scala/xiangshan/frontend/bpu/ittage/Ittage.scala:185-186 "Use the misprediction bits from the training bundle so allocation and certain updates (e.g. alt-provider penalty) only happen on actual mispreds." | Ittage.scala:338-346, 351-355 | docs/ittage_algorithm.md:209 "useful은 altDiffers 시에만 갱신된다"
  - status: proposed

- [R-ITTAGE-4] 오예측이 발생하면 (provider가 정답을 갖고 있었으나 자신이 없어 양보한 경우를 제외하고) 반드시 provider보다 긴 history 테이블 중 비어 있고 useful하지 않은 자리에 실제 target을 가진 새 엔트리가 만들어져야 한다. 그 시점에 useful한 엔트리를 덮어써서는 절대 안 된다.
  - evidence: src/main/scala/xiangshan/frontend/bpu/ittage/Ittage.scala:294-295 "Create a mask fo tables which did not hit our query, and also contain useless entries and also uses a longer history than the provider" | Ittage.scala:372-373, 378-387 | docs/ittage_algorithm.md:126
  - observe: ittage_allocate
  - status: proposed

- [R-ITTAGE-5] 자리가 없는 할당 시도가 누적되어 useful 초기화가 요청되면, 예측 요청이 매 cycle 들어오고 학습이 계속되는 부하에서도 모든 테이블의 모든 useful 표시가 반드시 유한한 시간 안에 지워져야 한다. 초기화 요청은 절대 무기한 미뤄지거나 잊혀서는 안 된다.
  - evidence: src/main/scala/xiangshan/frontend/bpu/ittage/IttageTable.scala:183 "Sweep one set index per cycle; all banks reuse resetSet so the full table resets in setsPerBank cycles." | IttageTable.scala:182-189 | docs/ittage_algorithm.md:285 "BPU가 매 cycle prediction을 발행하면 io.req.fire가 거의 항상 true → sweep이 사실상 진행 안 됨"
  - observe: ittage_reset_u, ittage_us_tick_reset
  - status: proposed

- [R-ITTAGE-6] 한 테이블의 한 set에 대한 쓰기(학습, 할당, useful 초기화)는 다른 set·다른 bank·다른 테이블의 엔트리를 절대 바꾸어서는 안 되며, 한 엔트리에 대한 쓰기는 그 쓰기가 갱신하기로 한 필드 외의 필드를 절대 바꾸어서는 안 된다.
  - evidence: src/main/scala/xiangshan/frontend/bpu/ittage/IttageTable.scala:190-194, 210-216 | src/main/scala/xiangshan/frontend/bpu/ittage/Bundles.scala:47 "Due to the bitMask the useful bit needs to be at the lowest bit" | IttageTable.scala:75 "split rows evenly across banks to allow predict/read and update on different banks."
  - status: proposed

- [R-ITTAGE-7] 학습으로 결정된 엔트리 쓰기(카운터 갱신, 할당, useful 초기화)는 predict read와 충돌하거나 쓰기가 몰리더라도 반드시 SRAM에 반영되어야 한다. 결정된 쓰기가 조용히 버려져서는 절대 안 된다.
  - evidence: src/main/scala/xiangshan/frontend/bpu/ittage/IttageTable.scala:197-198 "Each bank owns its own buffer so an update to bank A can proceed while bank B is serving a read." | IttageTable.scala:212, 276 (ittage_table_update_drop) | IttageTable.scala:237 "We do not want write request block the whole BPU pipeline"
  - status: proposed
