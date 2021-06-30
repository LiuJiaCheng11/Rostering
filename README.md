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
> #### 注：核算成加班的时长&核算成存假的时长 都是排班用户实际从此班次获得的，两者没有冲突，可以同时不为0，但不能同时为0(没有意义)
> #### 核算成加班的时长--一般会变成用户的薪资, 核算成存假的时长--一般会变成用户的存假
> #### 唯一约束:PlanItemId(实际在ObjectId里面)-Category-Effective=true

### 能产生记录的途径，目前有5种
### 一、班表在定义时就设置的加班时长和存假时长
>> #### 产生和删除的时机：排班数据的保存时产生，排班数据的删除时删除
>> #### 数据特征：ObjectId = PlanItem${PlanItemId}, Category = 1, OvertimeHours = 0, ActualHours = shift.Overtime, StoredVacationHours = shift.StoredVacation, Effective = 1, Status = 审批通过
>> #### 状态变更：随班表（PlanItem）变更，PlanItem是草稿->Effective = 0，PlanItem不是草稿->->Effective = 1

### 二、排班安排页面点击的加扣班
>> #### 产生和删除的时机：排班人加扣班保存时产生；加扣班撤销时删除，排班数据的删除时删除
>> #### 数据特征：ObjectId = PlanItem${PlanItemId}, Category = 0, OvertimeHours = [加扣班时长], ActualHours = [核算成加班的时长], StoredVacationHours = [核算成存假的时长], Effective = 1(如果当前PlanItem状态为草稿时，则为0), Status = 审批通过
>> #### 状态变更：随班表（PlanItem）变更，PlanItem是草稿->Effective = 0，PlanItem不是草稿->->Effective = 1
>> ##### 注：加扣班时长为用户当天上班时实际加扣班时长，比如早退2小时，加班2小时；现在核算成0.5小时加班和1小时存假。(`注意此公式不成立 OvertimeHours = ActualHours + StoredVacationHours`)

### 三、排班用户保存加班申请
>> #### 产生和删除的时机：流程保存加扣班申请时产生；流程物理删除时删除
>> #### 数据特征：ObjectId = Overtime${OvertimeId}, Category = 0, OvertimeHours = [申请人填写], ActualHours = [审批人换算], StoredVacationHours = [审批人换算], Effective = 0, Status = [流程是否执行审批通过]
>> #### 状态变更：流程方面：审批通过后Effective = 1，Status=[审批通过]，流程拒绝、终止、标记删除等Effective=0, Status=[撤销]；排班数据方面：排班数据的删除、加扣班撤销Effective=0, Status=[撤销]。撤回班表，Effective=0。发布班表，Effective=1
>> ##### 注1：关于唯一约束，主要针对排班人点击加扣班与用户申请加扣班的冲突校验，规定对于一个PlanItem而言，审批中和审批通过的Overtime的Record只有一个。即提交了申请后排班人不能再加扣班，排班人加扣班后不能再提交申请
>> ##### 注2：只有Effective = 1的加扣班Record才会加算加扣班时长，并且显示在班表
>> ##### 注3：实际判断加扣班是否生效还是看Effective，那么Status的作用是什么？只有1个作用，就是准备把Effective设成1时，必须检查Status的状态为[审批通过]。

### 四、排班月统计
>> #### 产生和删除的时机：编辑“排班月统计”后，归档后生效。当ActualHours=0且StoredVacationHours=0时，删除
>> #### 数据特征：ObjectId = MonthlyResult, Category = 0, OvertimeHours = [本月工作时长 - 本月应工作时长], ActualHours = [操作人换算], StoredVacationHours = [操作人换算], Effective = 1, Status = [审批通过]
>> #### 状态变更：无

### 五、积存假过期
>> #### 产生和删除的时机：时间流逝或人为因素导致积存假过期
>> #### 数据特征：ObjectId = Expiration, Category = 1, OvertimeHours = 0, ActualHours = 0, StoredVacationHours = -x（x为过期小时数）, Effective = 1, Status = [审批通过]
>> #### 状态变更：无

#### 操作总结如下：
#### 保存排班数据：产生【一】(若Shift的加扣班=0并且存假=0时不会产生)，Effective=是否已发布?1:0
#### 删除排班数据：删除【一】【二】，【三】的Effective=0，Status=[撤销]
#### 发布班表：【一】【二】的Effective=1，【三】的Effective= Status==[审批通过]?1:0
#### 撤回班表到草稿状态（现没有此操作）：【一】【二】【三】的Effective=0
#### 添加加扣班：产生【二】，Effective=是否已发布?1:0
#### 撤销加扣班：删除【二】，【三】的Effective=0，Status=[撤销]（此操作取消了）
#### 加班流程保存草稿：产生【三】，Effective=0，Status=[草稿]
#### 加班流程提交草稿：【三】Effective=0，Status=[审批中]
#### 加班流程执行流程拒绝、终止、标记删除等：【三】Effective=0，Status=[撤销]
#### 加班流程审批通过(Pass动作)：【三】Status= origin.Status==[审批中]?[审批通过]:origin.Status，Effective= Status==[审批通过]?1:0
#### 加班流程执行流程物理删除：删除【三】
#### 工作单元管理员核算排班月统计并归档，产生【四】
#### 归档后，工作单元管理员核算排班月统计，更新【四】
#### 时间流逝或人为因素导致积存假过期，产生【五】，如积存假过了系统设置的期限，仍未使用；或因其他操作，产生应该过期的积存假，详见积存假过期逻辑。

## 存假的逻辑
#### 数据库表
#### rostering_store_vacation 存假表
> #### 主要字段有
> #### 排班用户Id(UnitUserId)
> #### 年份(Year)
> #### 积存假余额(Balance)
> #### 积存假余额(单位:小时)(BalanceHours)
> #### 结转的天数，含小数部分(CarryOverDays)
> #### 结转的小时数(CarryOverHours)
> #### 本年可用积存假小时数[虚拟字段NotMapped]（TotalHours）
> #### 该年已同步到请假模块的总天数的整0.5部分(SyncDays)
> #### 注：每日工作时长不一定是8小时(可能7.5)，换算BalanceHours到Balance时会存在多位小数的情况，最终跑回界面上的数据是保留小数点后两位
> #### 唯一约束：Year-UnitUserId

>> #### 产生的时机：产生了相应年份的Record，且Record的StoredVacationHours ≠ 0
>> #### 数据特征：BalanceHours = [实时统计Record表中本年的数据]，CarryOverHours = [ISNULL(上一年的StoredVacation.BalanceHours, 0)]
>> #### 状态变更：无

### 实时结转机制：
##### ·当修改去年积存假时，会实时修改今年的年休假的结转小时数

### 积存假过期：
##### ·时间流逝可导致积存假过期
##### ·积存假发生变化时触发积存假过期检查，可导致积存假过期
##### ·采用90天前那天到未来获得的所有积存假小时数之和（不统计扣减的Record），不能大于【本年可用积存假小时数】
##### 详细逻辑：
##### 1、积存假过期周期设置n（n＞0）天
##### 2、【过期逻辑】：系统会统计n天前（包括那天）至未来所有获得的积存假小时数之和【n天前至未来统计小时数】，减去用户【本年可用积存假小时数】，结果为m，若m＞0，则过期m小时
##### 3、定时任务会执行【过期逻辑】
##### 4、一些导致增加【n天前至未来统计小时数】而不等量增加【本年可用积存假小时数】的操作会执行【过期逻辑】，如：撤销扣班，增加90天前的加班等
##### 积存假过期逻辑参考> https://my.oschina.net/ogurayui/blog/4989278

### 积存假超预支：
##### ·积存假发生变化时触发积存假超预支检查，当积存假超预支时，操作回滚
##### ·根据实际使用场景，用户可以提前使用未来的积存假，导致本年可用积存假小时数可能为负数（预支），但有个上限值
##### 详细逻辑：
##### 1、最大可预支积存假小时数设置为X，X为正数，表示最大预支X小时，即积存假最小允许-X
##### 2、【检查时机】：当积存假减少且值为负数时，超出预支值后返回错误信息，并回滚操作
##### 3、积存假过期不可能导致负数出现，所以不满足【检查时机】的条件
![积存假立即过期和超预支检查逻辑图](https://github.com/LiuJiaCheng11/Rostering/blob/master/img/stored-vacation-expiration.png)


### 同步到请假模块：
##### ·请假模块已内置一个编码为"StoreVacation"的假期，请假开放接口修改用户存休假；
##### ·排班和请假都各自管理其存休假数据以及结转事宜，虽然排班管理其存休假数据时需要受请假同步成功与否的影响，但不是强相关。
##### 详细逻辑：
##### 1、系统配置项启用了结转到请假模块：
##### 2、定时任务：把BalanceHours的整数部分（其实是整0.5的部分，因为请假天数最小单位是0.5）同步到请假模块，把整数部分的值写到该用户本年StoreVacation的记录中，下次执行定时任务时，需要和上次已同步的整数部分做对比，进行增加减少存休假的同步
##### 3、伪代码：下面[XXX]，均指取不大于当前值的最大的0.5的倍数的值(去尾法精确到0.5)
```
decimal days = [Sum(Record) + CarryOverDays] - SyncDays; //Sum(Record)是本年的所有产生积存假记录的和，CarryOverDays在下面会讲，固定值，SyncDays是已同步请假的整数部分，可正负
bool res = 请假同步("存休假编码", days);
if(res)
{ 
    storeVacation.SyncDays += days;
}
else
{
    //请假模块的可用天数不能为负，所以可能失败，比如用户把获得的存假用光了，此时发现他之前没上哪几天班(Sum(Record)减小)，要扣回去(days为负)，却没得扣了(res为false)
    //此时应该等此用户下次产生存假(Sum(Record)增大) ，days增加，同步就可能成功了
}
```
