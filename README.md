# 排班逻辑


## 加扣班/存假记录的逻辑
#### 数据库表
#### rostering_shift_extra_record 加扣班/存假记录表
> #### 主要字段有
> #### 排班用户Id(UnitUserId)
> #### ObjectId 里面有排班数据Id(PlanItemId)，目前有Overtime${OvertimeId}和PlanItem${PlanItemId}两种格式
> #### 类型(Category)
> #### 加扣班时长(OvertimeHours)
> #### 核算成加班的时长(ActualHours)
> #### 核算成存假的时长(StoredVacationHours)
> #### 是否有效(Effective)
> #### 审批状态(Status) 
> #### 注：核算成加班的时长&核算成存假的时长 都是排班用户实际从此班次获得的，两者没有冲突，可以同时不为0，但不能同时为0(没有意义)<br >
> #### 核算成加班的时长--一般会变成用户的薪资, 核算成存假的时长--一般会变成用户的存假
> #### 唯一约束:PlanItemId(实际在ObjectId里面)-Category-Effective=true(实际是Status=审批中和审批通过)

### 能产生记录的途径，目前有3种
### 一、班表在定义时就设置的加班时长和存假时长
>> #### 产生和删除的时机：排班数据的保存时产生，排班数据的删除时删除
>> #### 数据特征：ObjectId = PlanItem${PlanItemId}, Category = 1, OvertimeHours = 0, ActualHours = shift.Overtime, StoredVacationHours = shift.StoredVacation, Effective = 1, Status = 审批通过
>> #### 状态变更：排班人发布班表后Effective = 1, 排班人撤回班表后Effective = 0

### 二、排班安排页面点击的加扣班
>> #### 产生和删除的时机：排班人加扣班保存时产生；加扣班撤销时删除，排班数据的删除时删除
>> #### 数据特征：ObjectId = PlanItem${PlanItemId}, Category = 0, OvertimeHours = [加扣班时长], ActualHours = [核算成加班的时长], StoredVacationHours = [核算成存假的时长], Effective = 0(如果当前PlanItem状态为发布后的状态时，则为1), Status = 审批通过
>> #### 状态变更：排班人发布班表后Effective = 1, 排班人撤回班表后Effective = 0
>> ##### 注：加扣班时长为用户当天上班时实际加扣班时长，比如早退2小时，加班2小时；现在核算成0.5小时加班和1小时存假。(`注意此公式不成立 OvertimeHours = ActualHours + StoredVacationHours`)

### 三、排班用户保存加班申请
>> #### 产生和删除的时机：流程保存加扣班申请时产生；流程物理删除时删除
>> #### 数据特征：ObjectId = Overtime${OvertimeId}, Category = 0, OvertimeHours = [申请人填写], ActualHours = [审批人换算], StoredVacationHours = [审批人换算], Effective = 0, Status = [流程是否执行审批通过]
>> #### 状态变更：流程方面：审批通过后Effective = 1，Status=[审批通过]，流程拒绝、终止、标记删除等Effective=0, Status=[撤销]；排班数据方面：排班数据的删除、加扣班撤销Effective=0, Status=[撤销]。撤回班表，Effective=0。发布班表，Effective=1
>> ##### 注1：关于唯一约束，主要针对排班人点击加扣班与用户申请加扣班的冲突校验，规定对于一个PlanItem而言，审批中和审批通过的Overtime的Record只有一个。即提交了申请后排班人不能再加扣班，排班人加扣班后不能再提交申请
>> ##### 注2：只有Effective = 1的加扣班Record才会加算加扣班时长，并且显示在班表
>> ##### 注3：实际判断加扣班是否生效还是看Effective，那么Status的作用是什么？只有1个作用，就是准备把Effective设成1时，必须检查Status的状态为[审批通过]。

#### 操作总结如下：
#### 保存排班数据：产生【一】(若Shift的加扣班=0并且存假=0时不会产生)，Effective=是否已发布?1:0
#### 删除排班数据：删除【一】【二】，【三】的Effective=0，Status=[撤销]
#### 发布班表：【一】【二】的Effective=1，【三】的Effective= Status==[审批通过]?1:0
#### 撤回班表到草稿状态（现没有此操作）：【一】【二】【三】的Effective=0
#### 添加加扣班：产生【二】，Effective=是否已发布?1:0
#### 撤销加扣班：删除【二】，【三】的Effective=0，Status=[撤销]
#### 加班流程保存草稿：产生【三】，Effective=0，Status=[草稿]
#### 加班流程提交草稿：【三】Effective=0，Status=[审批中]
#### 加班流程执行流程拒绝、终止、标记删除等：【三】Effective=0，Status=[撤销]
#### 加班流程审批通过(Pass动作)：【三】Status= origin.Status==[审批中]?[审批通过]:origin.Status，Effective= Status==[审批通过]?1:0
#### 加班流程执行流程物理删除：删除【三】


## 存假的逻辑
#### 数据库表
#### rostering_store_vacation 存假表
> #### 主要字段有
> #### 排班用户Id(UnitUserId)
> #### 年份(Year)
> #### 积存假余额(Balance)
> #### 积存假余额(单位:小时)(BalanceHours)
> #### 该年已同步到请假模块的总天数的整0.5部分(SyncDays)
> #### 结转的天数，含小数部分(CarryOverDays)
> #### 注：每日工作时长不一定是8小时(可能7.5)，换算BalanceHours到Balance时会存在多位小数的情况，最终跑回界面上的数据是保留小数点后两位
> #### 唯一约束：Year-UnitUserId

>> #### 产生的时机：发生第一次的更新，但查询时是left join，故无数据也能查询
>> #### 更新的时机1：当产生Effective = 1 and StoredVacationHours <> 0 的record_shift_extra_record表的数据时，都会触发相关的积存假计算，计算为统计对应年份的Record的StoredVacationHours加算到StoreVacation的BalanceHours和换算成响应的Balance
>> #### 更新的时机2：当同步到请假模块时，会修改同步相关字段，如：SyncDays
>> #### 更新的时机3：发生结转事件时，此时间可自定义(不规定1月1日)，修改结转相关字段，如：CarryOverDays
>> #### 更新的时机4：当修改的Record的年份小于当前时间的年份时，立即触发结转事件
>> #### 更新的时机5：（暂无，未来可能有）每月结算，排班人在加扣班时只给了加扣班时长，然后每月月底时，查询这些加班时长并直接转化成[核算成加班的时长]或[核算成存假的时长]的数据(预计思路:生成第四种Record，不关联PlanItem)
### 以下为同步到请假模块以及结转的完整逻辑
>> #### ·请假模块已内置一个编码为"StoreVacation"的假期，请假开放接口修改用户存休假；
>> #### ·排班和请假都各自管理其存休假数据以及结转事宜，虽然排班管理其存休假数据时需要受请假同步成功与否的影响，但不是强相关。
>> #### 
>> #### 简要逻辑：
>> #### 在排班模块有两个表，ShiftExtraRecord（加扣(班/存假)的记录，以下简称Record），StoredVacation（用户年度存假记录）
>> #### 1、Record的产生：每次排班审批通过、不通过，加班申请等操作，都会产生加扣班或者加扣存假的记录Record
>> ####  2、排班定时任务：将发生日期在本年的Record的存假天数求和，并把求和的整数部分（其实是整0.5的部分，因为请假天数最小单位是0.5）同步到请假模块，把整数部分的值写到该用户本年StoreVacation的记录中，下次执行定时任务时，需要和上次已同步的整数部分做对比，进行增加减少存休假的同步
>> #### 3、存休假的结转：结转时，需要将上年的未同步到请假模块的存休假转移到今年来，当今年发生定时任务时，就能把去年还没同步的存休假给同步了
>> #### 4、结转后修改去年Record：修改完record后立即执行结转，等今年的定时任务来补偿/扣除存休假
>> #### 
>> #### 
>> #### 详细逻辑：
>> #### 1、Record的产生：同上，注意Record的天数是有0.2，0.3这样的小数的，程序中用了小时数/日工作小时计算
>> #### 
>> #### 2、排班定时任务：下面[XXX]，均指取不大于当前值的最大的0.5的倍数的值(去尾法精确到0.5)
>> ####   decimal days = [Sum(Record) + CarryOverDays] - SyncDays; //Sum(Record)是本年的所有产生积存假记录的和，CarryOverDays在下面会讲，固定值，SyncDays是已同步请假的整数部分，可正负
>> #### 
>> ####   bool res = 请假同步("存休假编码", days);
>> #### 
>> ####   if(res)
>> ####   { storeVacation.SyncDays += days; }
>> ####   else
>> ####   { 
>> #### 
>> ####     //请假模块的可用天数不能为负，所以可能失败，比如用户把获得的存假用光了，此时发现他之前没上哪几天班(Sum(Record)减小)，要扣回去(days为负)，却没得扣了(res为false)
>> #### 
>> ####     //此时应该等此用户下次产生存假(Sum(Record)增大) ，days增加，同步就可能成功了
>> #### 
>> ####   }
>> #### 
>> #### 3、存休假的结转：
>> ####   StoreVacation sv1; //2020年的，从数据库查
>> ####   StoreVacation sv2; //2021年的，去数据库查，没有的就新创建
>> ####   sv2.CarryOverDays = Sum(2020的Record) - sv1.SyncDays; //CarryOverDays是未结转天数，含小数，可正负
>> #### 
>> #### 4、结转后修改去年Record：修改以前年份的数据时，会逐年结转，直到把欠/补的天数正确结转到今年，由今年的定时任务修正，结转逻辑参考3



