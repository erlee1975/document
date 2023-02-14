# Configuration

## Video encoder 및 ROI 관련 configuration access flow

### 기존 방식

```plantuml
@startuml
skinparam handwritten true

participant "Web(관리자)" as admin

box "IP Cam" #LightBlue
participant "FCGI for param" as param_fgcid
participant "message bus" as wwmsgbus
participant "manager" as managerd
participant "stream pool" as strmpool
participant "config file" as config_file
participant "wwstreamd" as wwstreamd
participant "ISP" as isp
end box

== booting ==
            strmpool<->config_file: video encoder 및\nROI 정보 요청
            strmpool->strmpool: validation
                note right strmpool
                    channel의 rate control이 vbr일 경우,
                    ROI 설정을 할 수 없다.
                end note
            strmpool->isp: video encoder 및 \nROI 설정

== Video encoder rate control 설정 변경 ==
admin->param_fgcid: video encoder의\nrate control 설정
    param_fgcid->wwmsgbus: video encoder\ndynamic 설정
        wwmsgbus->managerd: video encoder\ndynamic 설정
            managerd->config_file: json file로 저장
                note right managerd
                    video encoder의 rate control이 vbr로 설정될 경우,
                    ROI를 적용할 수 없기 때문에, 기존 설정의 ROI 설정도 검사후 함께 갱신되어야 한다.
                end note
        wwmsgbus->strmpool: video encoder\ndynamic 설정
            strmpool<->config_file: ROI 정보 요청
            strmpool->strmpool: validation
                note right strmpool
                    vbr 정보를 사용하여 ROI 정보의 유효성을 확인해야 한다.
                end note
            strmpool->isp: video encoder\ndynamic 설정
                note right strmpool
                    ROI 정보와 함께 설정.
                end note

== ROI 설정 변경 ==
admin->param_fgcid: ROI 정보 요청
    param_fgcid->config_file: rate control 정보 요청
    param_fgcid->param_fgcid: validation
        note right param_fgcid 
            rate control 정보를 사용해 ROI 정보의 유효성을 확인해야 한다.
        end note
admin->param_fgcid: ROI 설정
    param_fgcid->config_file: rate control 정보 요청
    param_fgcid->param_fgcid: validation
        note right param_fgcid 
            rate control 정보를 사용해 ROI 정보의 유효성을 확인해야 한다.
        end note
    param_fgcid->wwmsgbus: ROI 설정
        wwmsgbus->managerd : json file로 저장
        wwmsgbus->wwstreamd: ROI 설정
            wwstreamd->isp: ROI 설정
@enduml
```

|개선 항목| 설명 |
|-|-|
|단일 책임 원칙| configuration의 validation의 책임이 여러 모듈에 걸쳐 중복 구현되고 있음.|
|처리 순서에 대한 보장 어려움| video encoder의 rate control을 설정할때 message bus에 의해 broadcasting된 message들로 인해 json file로 저장되는 동시에 ISP로의 설정 변경도 함께 시도 된다. ISP 설정 전에 ROI 값을 읽어와야 하는데 이때 vbr의 설정값이 반영되어 validation이 끝난 값인지 확인할 방법이 없기 때문에, 또다시 validation을 수행 해야한다.|

### 신규 방식(검토중)

```plantuml
@startuml
skinparam handwritten true

participant "Web(관리자)" as admin

box "IP Cam" #LightBlue
participant "FCGI for param" as param_fgcid
participant "stream pool" as strmpool
participant "config manager" as config_file
participant "ISP" as isp
end box

== booting ==
    strmpool->config_file: subscribe
        note right strmpool
            video encoder와 ROI관련 변경이 발생할 경우,
            notification 발생 요청
        end note
    strmpool->config_file: 설정 정보 요청
    strmpool->isp: video encoder와 ROI 관련 설정

== Video encoder 또는 ROI 설정 변경 ==
admin->param_fgcid: 설정 변경 요청
    param_fgcid->config_file: 저장 요청
            config_file->config_file: validation
                note right config_file
                    validation 수행한 후에 저장
                end note
        strmpool<--config_file: notification
            note right strmpool
                video encoder 또는 ROI관련 변경이 발생
            end note
        strmpool->isp: video encoder 및 ROI\ndynamic 설정
@enduml
```

|개선 항목| 설명 |
|-|-|
|단일 책임 원칙| configuration의 validation의 책임은 config manager에게 부과한다.|
|처리 순서에 대한 보장 어려움| 설정 변경 요청 message는 broadasting 되지 않으며, 오직 config manage 만 받아서 처리한다. 다른 모듈들은 설정 정보중에서 관심이 있는 항목들에 대해 구독을 하여 변경사항을 통지 받게 한다. 이렇게 하면, 설정 요청 -> validation -> 저장 -> notification이 순차적 발생하도록 보장할 수 있게 된다. |

## 구현 순서

모든 구현에 대해 refactoring을 한번에 진행할 수는 없기 때문에, 점차적인 진행을 해야한다.
이를위해, configuration manager를 먼저 별개의 process로 구현하고, 기 구현되어 있는 모듈들에 점차적으로 적용하도록 한다.

## Configuration Manager

