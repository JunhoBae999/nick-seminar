# File System - About Journaling
# 0. Intro
 [Introduction to Transactions](https://wiki.daumkakao.com/display/SolutionDelivery/Introduction+to+Transactions)  에서 다루었던 transaction, data update에 대한 신뢰의 문제
어플리케이션,DB 등등에서 transaction을 잘 관리하여 디스크에 Data를 write 하려고 하는데, 만약 중간에 전원이 내려간다면? 반만 쓰다가 내려간다면? 
*파일 시스템에 있어서 중요한 점은 바로 persistent 해야 한다는 것*
* 디스크의 데이터를 업데이트 하는 중에 전원이 나간다면?
* OS의 버그나 크래시가 나타난다면?
*현재 파일시스템에서는 어떻게 데이터 update를 안전하게 관리할까?*
file system의 crash-consistency problem
*How?*
*  fsck
*  journaling (ext3)
*  other
# 1. FSCK, Journaling
### 1.1 FSCK
* ‘file system check’, Unix/리눅스 파일 시스템 검사 수리 명령어,  현재는 확장버전인 e2fsck로 사용
* inconsistency를 감지하고 고치는 Unix 툴, 다른 시스템에서도 비슷한 툴이 존재함
* 전체 메타 데이터를 확인하고 복구를 수행하기 때문에 *매우 긴 시간이 소요됨*
### 1.2 Journaling
[image:EEEA6A4C-C9A5-4E44-BC64-598F71F73484-616-0000000C87DA8EB7/Attachment.png]
* data journaling
* ordered journling (default in ext3/4)
* write-back
### 2.1 Detailed Example
*2.1.1 Simple File System 가정*
[image:AAEC66DC-12F4-4A24-8E23-38DD085225B3-616-0000000C87D846D1/Attachment.png]
* bitmap (inode/data) : 8bit
* inodes : 8개
* data blocks : 8개
*2.1.2 예시 시나리오*
* 기존에 존재하는 파일에 하나의 data block을 추가하려는 상황 
* 현재 2번 inode가 할당되어 있고, 데이터 block은 4번에 할당되어 있는 상황
* inode
	* owner : nick.bae
	* permissions : read-write
	* size : 1
	* pointer : 4
	* pointer : null
	* pointer :  null
* 간단하게 size는 1이고, pointer에서 4번 Datablok을 가리키고 있음 (Da)
* Data block을 하나 추가해야 하는 경우, 디스크에서는 3개의 데이터를 업데이트 해야함
1. Inode : 새로운 block pointer를 지정해야 하고, 데이터가 추가되어야 하기 때문에 size를 update 해야 함
2. data block  : 새로운 데이터 블록 (Db)
3. data bitmap : 새로운 데이터 블록이 할당되었음을 표시해야 함
*2.1.3 정상 동작 결과*

[image:932FD56C-4DD8-4CEA-9CEE-2CBD2A76AC1C-616-0000000C87D2ADE8/Attachment.png]
* inode
	* owner : nick.bae
	* permissions : read-write
	* size : 1
	* pointer : 4
	* pointer : 5
	* pointer :  null
* updated bitmap :00001100
* updated data block: Db
총 3가지 disk write operation이 성공적으로 수행되어야 함. 하지만, wite()  system call을 날렸을 때 이 세가지 Operation이 즉시 이루어지지 않음
처음에는 Main Memory (page cache, buffer cache)에 남아있다가 디스크에 쓰기로 결정되고 나면 파일 시스템이 디스크에 write 요청을 보내게 됨
하지만*, 만약 업데이트 도중 crash가 발생한다면?*  한개만 수행되었는데 crash가 발생한다면? 두개가 수행되었는데 crash가 발생한다면?
*2.1.4 Crash Scenarios*

① Data block만 write한 경우
* 데이터는 디스크에 반영이 되었지만,  그 데이터를 포인팅 하는 Inode, bitmap이 존재하지 않음. 즉, 해당 블록이 할당된 사실 자체를 아무도 모름
ㄴ write 자체가 일어나지 않은 것과 동일
ㄴ file system crash consistency의 입장에서는 문제가 없음
ㄴ 물론 사용자에게는 데이터 loss
② Inode만 write한 경우
* node는 5번 주소를  포인팅 하지만 실제 데이터는 할당되지 않음. 이 경우, 가비지 데이터를 일게 됨.  또한 bitmap과 inconsistent한 상황이 발생함
③ Bitmap만 업데이트 된 경우
* inode에서는 5번 block을 가리키지 않기 때문에,  inconsistent한 상황에 빠짐. 즉, 5번 블락은 영원히 다시 쓰이지 않게 됨
ㄴ space leak

① inode와 bitmap만 write, data write은 Fali 한 경우
* 파일 시스템의 consistency 관점에서는 정상적인 상황. 하지만,  5번 블락에는 garbage 데이터가 존재
② Inode, Data block만 업데이트 되고 bitmap은 업데이트 하지 않은 경우
* Inconsistency 발생
③ Bitmap, data block은 업데이트 되고, inode는 업데이트 되지 않은 경우
* Inconsistency 발생, 해당 데이터가 어디에 쓰이는지 알 수가 없음

위에서 발견된 문제점들은 다음과 같음
* file system inconsistency
* space leaks
* garbage data return
따라서, file system을 하나의 상태에서 다른 상태로 전이시킬 때 atomic해야 함.  디스크는 한번에 하나의 wirte만 commit 하기 때문에, 그리고 Power loss나 system crash는 언제 발생할지 모르기 때문에,  대책이 필요함
*Crash-Consistency problem ( consistent-update problem)*
*2.1.5 Solution#1 The File System Checker*
초기 파일 시스템에서는 crash consistency에 대해 간단한 접근 선택, *inconsistency 발생을 허락한 후* 그 후에 고치는 방법
* fsck 툴
* Unix에서 inconsistency를 감지하고 고치는 Unix 툴, 다른 시스템에서도 비슷한 툴이 존재함
* 단, 모든 문제를 고치지는 못함 -> 메타 데이터만 consistent를 고려하기 때문
    * inode에는 기록되고 data block write는 실패한 경우와 같은 상황에서는 고치지 못함. 여전히 garbage data read
1. Superblock 확인
2. Free blocks 확인 
3. Inode State 확인
4. Inode Link 확인
5. Duplicates 확인 : 서로 다른 Inode가 같은 data block을 포인팅 하는지 등 확인 
6. Back blocks
7. Directory check


*단점*  : *너무 느림.*전체 디스크를 전부 살펴보면서 확인해야 하기 때문에. 대용량 디스크 환경, RAID 환경 등, 너무 느림
*2.1.6 Solution#2 Journaling (Write-Ahead Logging)*
*1) Concept*
* Consistent update problem의 문제를 해결하는 방법은, DBMS의 *write-ahead logging* 컨셉을 도입하는 것.
* File system에서는 이를 *write-ahead logging journaling* 명명
*2) Basic Idea*
디스크를 업데이트 할 때, 디스크에 바로 쓰기 전에 먼저 무엇을 할 것인지에 대한 note를 디스크 어느 공간에 남겨 놓는 것. 
무엇을 할지에 대한 노트를 남기면서,  write 도중 crash가 발생하더라도 무엇을 하려고 했는지 확인하고 다시 시도할 수 있음. 
* fsck처럼 전체 디스크를 전부 확인하지 않고, 무엇을 고쳐야 할 지 알 수 있음
* 저널링에서는 복구 시간에 걸리는 시간을 크게 감소시키기 위해, update 직전에 아주 약간의 작업이 추가됨. 
*3) Ext3*
*리눅스에서 journaling이 도입된 file system. *
* ext2 on-disk structure
[image:3A4B0640-B784-41D7-B5C4-8E7DD7D16382-616-0000000C87CF5910/Attachment.png]
* ext3 on-disk structure
[image:E550D51D-76AF-47C4-9D4F-F98EB0A940E0-616-0000000C87CE4F84/Attachment.png]
구조는 거의 동일한데, ext3에서는 Journal이라는 공간이 추가가됨. (별도의 디바이스에 존재할 수도 있고, 파일 시스템 내부의 파일로 존재할 수도 있음)
*4) Data Journaling*
이전의 파일을 추가하는 예시에 journaling을 도입한다면, 다음과 같은 프로세스를 거치게 됨
1. Journal Write : Transaction-begin /end block, 대기중인 데이터와 메타데이터의 변경 사항을 로그에 write가 완료될 때 까지 기다리는 것  
2. Checkpoint : 대기중인 데이터, 메타데이터를 파일 시스템에 업데이트 하는 것
① Journal Write
우선, 디스크에 실제로 wirte 하기 전에 journal이라는 log 공간에 우선 기록을 남김
[image:BD2CA974-70FD-4973-8FA1-6FB81F35632E-616-0000000C87C33D46/Attachment.png]
a. Transaction begin : Transition Descriptor Block
* 위의 TxB 공간
* 지금 Pending중인 Update에 대한 정보를 담고 있음. Inode의 주소, Bitmap의 주소, Data block의 주소, *Transaction Identifier(TID)* 
b. Data
* 위의 I[v2], B[v2], Db
* 실제 블록에 대한 정보를 담고 있음 <- physical logging (실제로 동일한 contents를 jpournal 공간에 똑같이 쓰는 것) 
* Logical logging : 실제 contents를 journal에 기록하는 것이 아닌, 논리적인 정보를 journal에 기록하는 것.
    * “지금 하려는 업데이트는 data block Db를 X라는 파일에 붙이려는 것이다”
    * 더 복잡하지만 공간을 절약하고 성능이 향상될 수도 있음
c. TxE (Transition Commit Block)
* 마지막 Block.
* transaction의 끝을 표시. TID를 담고 있음
② Checkpoint
* 위의 트랜잭션이 디스크에 안전하게 들어오면, 이제 file system에 반영할 수 있음.  위의 예시에서는 I[v2], B[v2], Db를 디스크에 반영시킴
*2.1.7 Journal Write 도중 crash가 발생한다면?*
Journal에 트랜잭션을 기록하기 위해 위에서의 TxB, I[v2], B[v2], Db. TxE를 write 하는 경우, 가장 간단한 방법은 하나씩 write 하는 것 하나가 끝나면 다음 write  수행 → *느림*
따라서, 한번에 위의 5개의 write를 한번에 처리해서 하나의 sequential write로 처리하면 더 빠름 → *but, unsafe*
* 만약 저렇게 한번에 big write를 요청하게 되면, *디스크 내부의 스케줄링을 통해서 big write의 작은 조각들을 아무 순서대로 쓸 수도 있기 때문*

1) TxB, I[v2], B[v2], TxE를 Write 
2) 이후 data block Db를 Write
 → 1) 수행 이후 power off 된다면 디스크의 Journal은 아래와 같은 상황이됨
[image:EBA41EA6-CA90-4EFE-B828-D52ADF77E5D7-616-0000000C87B3796F/Attachment.png]
* 트랜잭션은 정상적으로 보일지 몰라도,  파일 시스템은 4번째 블록을 알 수가 없고 잘못된지도 모름 . 임의의  (그 자리에 남아있는)데이터
* 이 상태에서, 시스템이 리부트 되고 리커버리를 돌릴 때, 위의 트랜잭션이 다시 실행될 것이고,  무지성으로 ??이라는 데이터를 Db가 들어갈 주소에 저장.
-> 만약 Critical한 파일 시스템의 조각이면 심각한 문제 발생

이를 방지하기 위해, File system은 transactional write를 *두 단계로 분리*
*1)  TxE block을 제외한 모든 블록을 저널에 한번에 write, 완료까지 기다림*
[image:FDA77C44-CAF6-43A1-8972-771FD2DB9C39-616-0000000C87AC5C80/Attachment.png]

*2) 1의 write이 완료가 되면, TxE block  write를 발행, journal이 safe state가 되도록 함*
*[image:99446D05-2774-4D64-BFFB-5E3A0429380C-616-0000000C87AA2670/Attachment.png]*
Disk에서 512-byte는 atomic하게 write하기 때문에, TxE가 atomic 한게 Write 되는걸 보장하려면 512-byte block으로 만들어야함.

1. Journal write : 트랜잭션의 Contents를 Log에 작성하고, 완료될 때 까지 기다린다. (TxB, metadata, data)
2. Journal commit : transaction commit block을 로그에 작성하고, 완료될 때 까지 기다린다. 완료가 되는경우 이를 commit되었다고 한다.
3. Checkpoint : 디스크의 원래 위치에 실제 업데이트를 반영한다.
*2.1.8 Recovery*
파일시스템이 어떻게 저널의 Contents를 사용해 crash 상황에서 recovery를 진행하는지
1 만약, 트랜잭션이 로그에 온전히 write 전에 crash가 발생한다면 (commit 전) pending updates는 무시된다.
2 만약, Commit 이후에 Crash가 발생한다면,  시스템이 부트할 때 로그를 스캔하고 커밋된 트랜잭션을 찾아 순서대로 재실행한다. 
* 가장 간단한 방법, Redo logging
* checkpointing 중에 Crash가 발생하게 되더라도 재실행으로 해결 가능
* 리커버리 프로세스가 일어나는 경우는 드물기 때문에, 중복 write가 발생하더라도 몇번 안일어나기 때문

*2.1.9 Batching Log Updates*
*하지만, 위의 basic protocol은 디스크 트래픽이 상당히 증가하게 됨*
*예시) 동일 디렉토리에 File1, Fille2를 만드는 경우*
* 하나의 파일을 만들기 위해선 최소 inode bitmap, 새로운 파일의 Inode, parent directory의 Data block, parent directory의 Inode를 update해야 함
* 저널링을 하게 될 경우, 위의 정보들을 하나의 파일을 만들 때마다 전부 저널링해야함. 따라서, 같은 블록을 계속해서 write하게 되는 상황이 발생
*  EXT3 등에서는 각각의 업데이트마다 디스크에 Commit하지 않고,  *global transaction에 모든 업데이트를 버퍼링하기도 함*
	*  파일이 만들어 질 때, 파일 시스템은 in-memory  inode bitmap, 파일의 inodes, directory data, directory inode를 dirty로 표시하고, 지금의 트랜잭션을 구성하는 블록 리스트에 추가
	* 시간이 되면 (가령 timeout이 5초라면) 이 하나의 global transaction을 commit
*2.1.10 Making the Log Finite*
*하지만, 로그의 사이즈는 한정적임. 만약 transaction을 계속해서 저널에 기록하다보면 journal이 다 차버림*
1) 로그가 길어질 수록, Recovery가 길어짐. 
*2) 로그가 가득 차면, 그 이후의 트랜잭션이 더이상 디스크에 커밋될 수 없음*
→  로그를 circular data structure로 관리, 재사용
→  journal 을 circular log라고 부르는 이유.
따라서, 파일 시스템은 checkpoint 이후에  공간이 재사용될 수 있도록 checkpoint된 transaction을 Free 해줘야 함 
가장 간단한 방법은, journal superblock을 두고 oldest and newest non-check pointer transaction을 관리
[image:B97DE92D-ABE6-433E-929C-48558525E59A-616-0000000C87A78787/Attachment.png]
* 로그 공간을 재사용할 뿐만 아니라 복구 시간 단축 가능
1. Journal write
2. Journal commit
3. Checkpoint
4. Free : journal super block을 업데이트 해서, 저널 안의 완료된 Transaction을 Free marking 한다.

*2.1.11 Metadata Journaling*
지금까지 계속해서 data block을 두번 씩 write 하고 있는데, (journal에 한 번, 실제 위치에 한 번) system crash라는 흔치않은 경우를 위해 비용이 너무 많이 듬 
*→ 즉, 복구시간은 빨라졌지만, normal operation이 너무 길어짐*
*따라서, 이 퍼포먼스를 향상 시키기 위해 나온 것이 바로 ordered jounaling (= metadata journaling)*    cf. 모든 데이터를 저널링 =  data journaling 
* 프로세스는 똑같은데, *user data가 Journal에 기록되지 않는다는 것*
[image:F363FFBA-BEC8-4868-B212-CF99FC1B0D3C-616-0000000C87A6ADEB/Attachment.png]
* Disk I/O 트래픽의 대부분은 사실 Data block이기 때문에, data를 기록하지 않는다면 트래픽을 상당히 줄일 수 있음

그러면, data block을 언제 디스크에 써야 할까?
파일 추가 예시에서 살펴보면, 업데이트는 I[v2], B[v2], Db를 업데이트 해야 함 
앞의 두개는 메타데이터이기 때문에 로깅이 될 것이고, check-point 됨 / Db는 파일 시스템에 한번만 write
만약, transaction이 끝나고 난 후 Db를 disk에 write한다면? I[v2]가 garbage data를 포인팅 하게 될 것이다. 
따라서, 보통 파일 시스템 (ext3)는 data block write를 가장 먼저 하고, 이후의 metadata를 저널링 한다.

1. Data write : 데이터를 최종 위치에 Write, 완료까지 기다린다. 
    * 사실 기다리는 건 Optional. 꼭 안기다려도 되고, 메타데이터 저널링이 동시에 이루어져도 괜찮다. 다만, 커밋 되기 전에만 (커밋 블락이 저널링 되기 전에만 <- step 3)  이루어지면 된다.
2. Journal metadata write : begin block과 메타데이터를 로그에 기록한다. write가 완료될 때 까지 기다린다.
3. Journal commit
4. Checkpoint metadata
5. Free 
*2.1.12 특이 케이스*
“What’s the hideous part of the entire system? … It’s deleting files. Everything to do with delete is hairy. Everything to do with delete… you have nightmares around what happens if blocks get deleted and then reallocated.”

Metadata jounraling을 사용하고 있는 상황. 
1. 사용자는 Foo라는 디렉토리에 어떤 파일을 추가함. 따라서, foo의 contents가 저널에 기록됨 (디렉토리는 메타데이터니까)
* 디렉토리 데이터가 1000번 주소에 있는 상황
[image:8500E582-1202-4C51-976C-AF1ED0738066-616-0000000C87A53341/Attachment.png]
2. 이때, 사용자가 foo를 전부 지워버려서 1000번 주소가 재사용될 수 있게 되었는데, 이 때 bar라는 새로운 파일을 만들고 이게 1000번 주소에 기록됨
[image:BEE95CE2-3AFB-4EB8-96C4-C5B4F5D3148F-616-0000000C87A34088/Attachment.png]
* 즉, Bar의  데이터는 디스크에 기록되고, Inode 역시 커밋됨. 하지만, 저널에는 bar의 데이터는 기록되지 않음
3. 이런 상황에서, checkpoint 이후 Free 되지 않은 상황에서 crash가 발생해서 recovery가 이루어지는 경우, 저널을 그대로 재실행하게 된다면 1000번 주소 (bar data, 저널링 되지 않음)를 foo 디렉토리 데이터(저널링 됨)로 덮어 씌워버리게 됨


체크포인트되어서 저널에서 나가기 전 까지는 해당 블록을 재사용하지 않거나, Ext3에서는 revoke Record를 남겨, 실행되지 않게 함
[image:8AEACB56-449C-4901-B0AE-072ADBDE0CD9-616-0000000C87A205F8/Attachment.png]


*2.1.13 Journaling Wrapup*
*1) Data Journaling Timeline*
*[image:0640692C-DCB3-430D-8F78-0F1A4A9ABA2D-616-0000000C87A0A4DF/Attachment.png]*
*2) Metadata Journaling Timeline*
*[image:6E1B4CD2-B49B-42BD-9429-A4D263FFC62E-616-0000000C879FC28C/Attachment.png]*
### 3. Other Solution
*3.1 Soft Update*
디스크 구조가 불일치 상태로 남아있지 않도록 모든 시스템에 대한 쓰기를 신중하게 진행.
Ex) 어떤 데이터 블록에 대한 inode를 업데이트하기 전에 해당 데이터 블록을 반드시 먼저 써줘서 garbage data 문제가 발생하지 않도록 할 수 있음.
* 단점 : soft update가 진행중인 도중 crash가 발생할 경우 데이터가 오염될 수 있음, 구현이 복잡함
*3.2 Copy-on-write(COW) - Brtfs*
COW는 파일이나 디렉터리를 절대 덮어쓰지 않으며 디스크에서 이전에 사용되지 않은 위치에 새 업데이트를 배치
여러 업데이트가 끝난 뒤 COW 파일 시스템은 새로 업데이트된 구조에 대한 포인터를 포함하도록 파일 시스템의 루트 구조를 뒤집음. 
### 4. MySql, Sqlite에서의 Journaling
*4.1 MySql*


* 디스크의 데이터 파일이나 인덱스 정보를 메모리에 캐시해두는 공간. 조회 성능 뿐만 아니라 쓰기 지연 기능까지 제공. 
* 구조 : 페이지 단위로 관리. LRU, 플러시 리스트, 프리 리스트
* InnoDB가 데이터를찾는 과정
1. 레코드가 저장된 데이터 페이지가 버퍼 풀에 있는지 검사
2. 없으면 디스크에서 필요한 페이지를 버퍼 풀에 적재, 해당 페이지에 대한 포인터를 LRU 헤더에 추가
3. LRU 헤더에 적재된 페이지가 실제로 읽히면 MRU 헤더 부분으로 이동
4. 버퍼 풀에 상주하는 데이터 페이지에는 사용자 쿼리가 얼마나 최근에 접근했는지에 따라 AGE가 부여. 오래된건 버퍼풀에서 제거됨. LRU로 이동

InnoDB에서는 데이터가 변경되면 리두 로그에 기록하고, 버퍼 풀의 데이터 페이지에도 반영. 따라서, 리두 로그는 특정 데이터 페이지와 연결. 이노디비에서는 체크포인트를 발생시켜 디스크의 리두로그와 데이터 페이지를 동기화시킴


* InnoDB 스토리지 엔진의 리두로그는 리두 로그 공간의 낭비를 막기 위해 페이지의 변경된 내용만 기록함. 
* 따라서, 더티 페이지를 디스크 파일로 플러시할 떄, 일부만 기록되는 문제가 발생하면, 복구할수 없을 수도 있음
* Partial-page, Torn-page (하드웨어 오작동, 시스템 비정상 종료)
*이에 대한 해결책으로 InnoDB에서는 Doulbe-write 기법 이용*
[image:DFC55727-594B-413E-8B87-74C5F5322CE1-616-0000000C879A5297/Attachment.png]
* A-D의 더티페이지를 디스크로 플러시 하는 경우
* 실제 데이터 파일에 write 하기 전에 A-D까지의 더티페이지를 묶어서, 한번의 write로 시스템 테이블 스페이스의 DoubleWrite 버퍼에 기록
* 이후, 각 더티 페이지를 파일의 적당한 위치에 하나씩 랜덤으로 씀
* 잘 write되면 DoubleWrite 버퍼 공간은 필요가 없으나, 중간에 실패하는 경우 사용
* 도중에 만약 운영체제가 비정상적으로 종료되는 경우, InnoDB 스토리지엔진은 재시작 될 때 항상 DoubleWrite 버퍼의 내용과 데이터 파일의 페이지들을 모두 비교해서, 다른 내용을 담고 있는 페이지가 있으면 해당 내용을 복사.
* HDD와 같이 자기 원판이 회전하는 저장 시스템에서는 어차피 한 번의 순차 디스크 쓰기를 하는 것이라 경우 부담이 없으나 SSD와 같은 랜덤 IO는 성능 이슈가 있을 수도.
* 무결성이 중요하다면 활성화, 성능이 중요하다면 innodb_doublewrite 시스템 변수로 OFF 가능
* InnoDB 리두 로그 동기화 설정을 1이 아닌 다른 값으로 한 경우 비활성화
만약, 무결성이 중요한 경우라면, DoubleWrite 뿐만 아니라 InnoDB 리두 로그와 복제를 위한 바이너리 로그 등 트랜잭션을 Commit 하는 시점에 동기화 할 요소들 주의
[장애와 관련된 XtraBackup 적용기 | 우아한형제들 기술블로그]( [https://techblog.woowahan.com/2576/](https://techblog.woowahan.com/2576/) ) 


*4.2 Sqlite*
1.  롤백(Roll-back) 모드 : 원본 데이터가 저장된 저널링 파일을 생성하여 원본 데이 터를 수정되면 해당 파일을 삭제하는 방식
2. WAL (Write-ahead Log) 모드 : 파일시스템에서 제공하는 저널링 기법과 유사하게 WAL 파일을 생성한 뒤 변경된 데이터를 WAL 파일에 먼저 반영하고 원본 데이터를 수정하는 방식
-> 변경된 데이터를 저장장치에 동기화하기 위 해 fsync를 빈번하게 호출함으로써 시스템의 I/O 성능을 저하시킨다는 단점이 존재한다. 
*4.3 참고)  MySql, Sqlite와 ext4에서의 중복 journaling으로 인한 성능 이슈*
[image:FD0AFB22-2DB2-47C2-9E96-EC365793FDBA-616-0000000C8797DB48/Attachment.png]
### -5.  추가) RAID에서 Journaling 오버헤드-
RAID(Redundant Array of Independent Disks) 
: 여러 개의 디스크에 데이터를 나눠 저장하는 기술
* RAID를 이루는 구조에 따라 다양 레벤 존재
1 RAID Linear JBOD (Just a Bunch of Disks)으로도 불 리며 2개 이상의 디스크를 1개의 볼륨으로 사용하는 방 식이다. RAID Linear에서는 데이터를 하나의 디스크에 순차적으로 할당하고 첫 번째 디스크가 완전히 채워질 때 다음 디스크를 채운다. 따라서 I/O 작업이 디스크 간 분할되지 않아 RAID 성능의 이점을 제공하지 못한다. 
* RAID 0은 디스크 스트라이핑으로 하나의 데이터를 여 러 드라이브에 분산 저장을 함으로써 빠른 입출력을 가 능하게 한다. 그러나 데이터의 분산 저장에만 초점이 맞 춰져 있어 스트라이프 되어 있는 디스크 중 1개만 장애 를 일으키더라도 데이터를 모두 유실할 위험성이 있다. 
* RAID 1은 디스크 미러링으로 하나의 디스크에 기록되 는 모든 데이터가 나머지 하나의 디스크에 고스란히 복 사되는 방법으로 저장하게 된다. 이 경우 2개의 디스크 중 1개가 장애를 일으키더라도 남은 1개의 데이터는 장 애를 일으킨 디스크의 데이터와 똑같기 때문에 안정성이 높다. 그러나 미러링의 특성 상 같은 데이터를 2번 쓰기 때문에 사용할 수 있는 용량이 50%로 제한된다는 단점 이 있다.
 
* RAID 4 RAID 0의 스트라이프 구조가 가진 이점을 유 지하면서 데이터 안정성을 위해 패리티 비트를 관리하는 디스크를 추가한 구조를 가지고 있다. 하지만 패리티 디스크에서 패리티 비트 전체를 관리하기 때문에 패리티 디 스크에 I/O가 몰리는 현상이 발생한다. 또한 패리티 디스 크가 손상됐을 시 데이터 복구가 어려운 문제점이 있다. 
* RAID 5는 패리티 비트를 모든 디스크에 분산해서 업 데이트 하는 구조이다. 이런 구조는 RAID 4의 단점인 패 리티 디스크 I/O 병목현상과 패리티 디스크 고장 시 데 이터 복구불가 문제를 해결할 수 있다.


# 6.  참고자료
* 이민호, “DBMS와 파일시스템의 중복된 저널링 오버헤드 분석”, 2017년 한국소프트웨어종합학술대회 논문집, 2p
* 백은빈,이성욱,Real MySQL 8.0(위키북스,2021.09.08), 108p
* 다케우치 사토루,실습과 그림으로 배우는 리눅스 구조(한및미디어, 2020)
*  [https://pages.cs.wisc.edu/~remzi/OSTEP/file-journaling.pdf](https://pages.cs.wisc.edu/~remzi/OSTEP/file-journaling.pdf) 


 











