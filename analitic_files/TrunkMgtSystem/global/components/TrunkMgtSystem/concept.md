# System concept in C4 Container

```plantuml
@startuml
!NEW_C4_STYLE=1
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml
SHOW_PERSON_OUTLINE()
' Tags support no spaces in the name (based on the underlining stereotypes, which don't support spaces anymore). 
' If spaces are requested in the legend, legend text with space has to be defined
AddElementTag("backendContainer", $fontColor=$ELEMENT_FONT_COLOR, $bgColor="#335DA5", $shape=EightSidedShape(), $legendText="backend container (eight sided)")
AddRelTag("async", $textColor=$ARROW_FONT_COLOR, $lineColor=$ARROW_COLOR, $lineStyle=DashedLine())
AddRelTag("sync/async", $textColor=$ARROW_FONT_COLOR, $lineColor=$ARROW_COLOR, $lineStyle=DottedLine())

' External Systems
System_Ext(Protei14, "Protei 1/4", "Software System", $sprite="system")
System_Ext(TMS, "TMS System", "Software System", $sprite="system")
System_Ext(TFOP, "ТФОП", "Software System", $sprite="system")
System_Ext(ExolveLKR, "Exolve LKR", "Software System", $sprite="system")
System_Ext(ExolveLKM, "Exolve LKM", "Software System", $sprite="system")
System_Ext(RDS_API, "RDS API", "Number Mgt System\n[Container: Golang]", $sprite="system")
System_Ext(NMS_API, "NMS API", "Number Mgt System\n[Container: Golang]", $sprite="system")

' System Boundary
System_Boundary(TrunkMgtSystemBoundary, "TrunkMgtSystem") {
    
    Container(TrunkMgtAPI, "Trunk Mgt API", "Container: Golang", "", $sprite="container")

    ContainerDb(TrunkMgtDB, "Trunk Mgt DB", "Container: PostgreSQL", "AccountID/TrunkID/ExtimalIP/Application(*)/\nSource/InternalIP/InternalPort(*)/\nDescription(*)/CPS(*)/SL(*)", $sprite="database")
    ContainerDb(TrunkAPIDB, "Trunk API DB", "Container: Redis", "Replica + sentinel", $sprite="database")
    Container(TrunkAPI, "Trunk API", "Container: Golang", "", $sprite="container")
    Container(SyncWorker, "Sync Worker", "Container: Golang/Spine", "", $sprite="worker")
    ContainerDb(SyncStorage, "Sync storage data", "Container: Redis", "Store task process data")
    ContainerQueue(DataForCache, "Data for cache", "Container: Apache Kafka/Redpanda", "")

    Rel(TrunkMgtAPI, TrunkMgtDB, "Operate with data", "")
    Rel(TrunkMgtAPI, DataForCache, "Added trunk data", "")
    Rel(TrunkMgtAPI, TrunkAPI, "Get data by IP", "")
    Rel(TrunkAPI, TrunkAPIDB, "Store trunc data", "")
    Rel(SyncWorker, DataForCache, "Added trunk data", "")
    Rel(SyncWorker, TrunkAPI, "Add numbers data", "")
    Rel(SyncWorker, SyncStorage, "Syncing progress", "")
}

' External Relationships
Rel(Protei14, TMS, "Call", "")
Rel(TMS, TFOP, "Call", "")
Rel(TMS, TrunkAPI, "Get data", "")

Rel(ExolveLKR, TrunkMgtAPI, "Operate data", "")
Rel(ExolveLKM, TrunkMgtAPI, "Operate data", "")

Rel(TrunkAPI, RDS_API, "Get LC by number", "")
Rel(SyncWorker, NMS_API, "get numbers by lc\n[GetCustomerNumbers]", "")

@enduml
```
