首先来看一段代码
```scala
class CorrectionProcessWindowFunc(val allowLateTime: Int) extends ProcessWindowFunction[OnlineRecognitionData, (String, Long, Int), String, TimeWindow] {

  private final val mapStateWindowDesc = new MapStateDescriptor[String, PidNode]("window-state",
    createTypeInformation[String], createTypeInformation[PidNode])
  private final val tailMapStateWindowDesc = new MapStateDescriptor[String, PidNode]("tail-window-state",
    createTypeInformation[String], createTypeInformation[PidNode])
  private final val countStateDesc = new ValueStateDescriptor[Int]("countState", TypeInformation.of(classOf[Int]))


  // 全量的
  private var mapState: MapState[String, PidNode] = _
  private var tailMapState: MapState[String, PidNode] = _
  private var countState: ValueState[Int] = _


  override def process(key: String, context: Context, elements: Iterable[OnlineRecognitionData], out: Collector[(String, Long, Int)]): Unit = {
//        if (mapState == null) {
//          mapState = context.windowState.getMapState(mapStateWindowDesc)
//        }
//        if (tailMapState == null) {
//          tailMapState = context.windowState.getMapState(tailMapStateWindowDesc)
//        }
//        if (countState == null) {
//          countState = context.windowState.getState(countStateDesc)
//          countState.update(0)
//        }
    mapState = context.windowState.getMapState(mapStateWindowDesc)
    tailMapState = context.windowState.getMapState(tailMapStateWindowDesc)
    countState = context.windowState.getState(countStateDesc)


   
  }

}


```
