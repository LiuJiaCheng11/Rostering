# 排班逻辑
## 加扣班/存假计算逻辑
#### 数据库表
#### record_shift_extra_record 加扣班/存假记录表
> #### 主要字段有
> #### 排班用户Id(UnitUserId)
> #### 排班数据Id(PlanItemId)
> #### 类型(Category)
> #### 加扣班时长(OvertimeHours)
> #### 核算成加班的时长(ActualHours)
> #### 核算成存假的时长(StoredVacationHours)
> #### 是否有效(Effective)
> #### 注：核算成加班的时长&核算成存假的时长 都是排班用户实际从此班次获得的，两者没有冲突，可以同时不为0，但不能同时为0(没有意义)<br >
> #### 核算成加班的时长--一般会变成用户的薪资, 核算成存假的时长--一般会变成用户的存假
> #### unique：PlanItemId-Category

### 能产生记录的途径，目前有3种
### 一、班表在定义时就设置的加班时长和存假时长，排班人排班发布后产生；发布后的排班数据的变更或删除
>> #### 产生和删除的时机：排班人发布班表和撤回班表
>> #### 数据特征：Category=1, OvertimeHours = 0, ActualHours = shift.Overtime, StoredVacationHours = shift.StoredVacation, Effective = 1
>> #### 状态变更：无

### 二、排班人在草稿状态时点击的加扣班
>> #### 产生和删除的时机：排班人加扣班和撤销加扣班；排班数据的变更或删除
>> #### 数据特征：Category=0, OvertimeHours = [加扣班时长], ActualHours = [核算成加班的时长], StoredVacationHours = [核算成存假的时长], Effective = 0(如果当前PlanItem状态为发布后的状态时，则为1)
>> #### 状态变更：排班人发布班表后Effective = 1, 排班人撤回班表后Effective = 0
>> 注：加扣班时长为用户当天上班时实际加扣班时长，比如早退2小时，加班2小时；现在核算成0.5小时加班和1小时存假。(`注意此公式不成立 OvertimeHours = ActualHours + StoredVacationHours`)

### 三、排班用户保存加班申请
>> #### 产生和删除的时机：流程保存加扣班申请；排班数据的变更或删除
>> #### 数据特征：Category=0, OvertimeHours = [申请人填写], ActualHours = [审批人或排班人后续转化], StoredVacationHours = [审批人或排班人后续转化], Effective = 0
>> #### 状态变更：审批通过后且关联的PlanItem为发布后的状态Effective = 1，一般申请时要发布后的PlanItem才是可视的，但可能申请提交后，排班人撤回来了, 其他时候Effective = 0

无论是哪种途径产生的数据，都必须要对应的班表发布后才能生效
