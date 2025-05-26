Staff earn Bank hours instead of Overtime. This can then be booked as Holiday under the Holiday type Banked Hours and will deduct for what you have earned. You are allowed to Adjust the amount to increase or decrease the total. The end of year amount can be carried to next year and can be capped for each company. This should automatically generate when the next year is created. I found many questions raised on the amounts and decided to run a comparison on the report. The hardest change was if it had been missed for a few years then I needed to keep transferring the balance through the middle years. I will admit I did not achieve this but was very useful in detecting when it did not add up for the following year so could discuss where the issues were. Note I can not show GUI as that is not my work, the report looks very similar to how it is shown in the table.
Here you can see the balance of 2022 did not transfer to 2023.

![image](https://github.com/user-attachments/assets/6ce93fbe-6004-4a46-82ca-ee9448fd135a)
 
Here I can see Mary has recorded more Banked hours than what is saved in this table.
 
![image](https://github.com/user-attachments/assets/fe8c3897-467b-42a8-87d1-f2df053277f4)

Bonus information is I am instructed not to add or alter the database permanently. In the script you will find a function that creates then drops at the end. This converts seconds into HH:MM and this can handle time over 100:00 and negative time values. I could not find any premade function that was able to work even though their documentation said it could. If you move the @unit varchar(20) = 'ss'; into the calling section you will be able to minutes and hours (need to remove the =’ss’ as I always work in seconds.



-- Notes the Unit has been moved from the 1st Declare and set to 'ss' all the time but can handle other times other than seconds

/****** Object:  UserDefinedFunction [dbo].[ConvertTimeToHHMMSS]    Script Date: 30/09/2024 09:14:03 ******/
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

create function [dbo].[ConvertAnyTime]
(
    @time decimal(28,3)
    
)
returns varchar(20)
as
begin
	

    declare @seconds decimal(18,3), @minutes int, @hours int,@unit varchar(20) = 'ss';
			if(@unit = 'hour' or @unit = 'hh' )
				set @seconds = @time * 60 * 60;
			else if(@unit = 'minute' or @unit = 'mi' or @unit = 'n')
				set @seconds = @time * 60;
			else if(@unit = 'second' or @unit = 'ss' or @unit = 's')
				set @seconds = @time;
			else set @seconds = 0; -- unknown time units

			--set @hours = convert(int, @seconds /60 / 60);
			--set @minutes = convert(int, (@seconds / 60) - (@hours * 60 ));
			--set @seconds = @seconds % 60;
	
if coalesce(@time,0)>=0
			set @hours = convert(int, @seconds /60 / 60)
if coalesce(@time,0)>=0
			set @minutes = CONVERT(int, (@seconds / 60) - (@hours * 60))
if coalesce(@time,0)>=0
			set @seconds = @seconds % 60;

if coalesce(@time,1)<0	
			set @hours = convert(int, @seconds /60 / 60);
if coalesce(@time,1)<0	
			set @minutes =CONVERT(int, (@seconds / 60) - (@hours * 60))-CONVERT(int, (@seconds / 60) - (@hours * 60))-CONVERT(int, (@seconds / 60) - (@hours * 60))
if coalesce(@time,1)<0	
			set @seconds = @seconds % 60;
	

    return 
        convert(varchar(9), convert(int, @hours)) + ':' +
        right('00' + convert(varchar(2), convert(int, @minutes)), 2) 
		--+ ':' +right('00' + convert(varchar(6), @seconds), 6)
	
end
GO

--Note this only works with adding the time function

IF object_ID ('tempdb..#TempBankedHours') IS NOT NULL
  DROP TABLE #TempBankedHours
go

Declare 
@year int = 2022,
@year2 int = 2025,
@minyear int = (select min(year(periodstart)) from BankedHours)

SELECT E.empref,E.payrollno as Payroll, E.forenames,
       CONVERT(VARCHAR(8), BH.PeriodStart, 3) AS PeriodStart,
	   CONVERT(VARCHAR(8), BH.periodEnd, 3) AS PeriodEnd,

       'Easy Read' as Converted,	
	   coalesce(NULLIF(dbo.ConvertAnyTime(BH.hrsbrghtfwd),'0:00'),'') as BrtFwd2,
       coalesce(NULLIF(dbo.ConvertAnyTime(BH.Banked),'0:00'),'') AS Banked2,
	   coalesce(NULLIF(dbo.ConvertAnyTime(HT.HoursTransfered),'0:00'),'') as HrsTransfer,
	   coalesce(NULLIF(dbo.ConvertAnyTime(BH.Adjustment),'0:00'),'') as Adjust2,
	   case when BH.absbooked = 0 then '' else dbo.ConvertAnyTime(coalesce(BH.absbooked,0)) END AS BankBook2,
	   dbo.ConvertAnyTime((coalesce(BH.Banked,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0) -(coalesce(BH.Absbooked,0)))) as DBtotal,
	   dbo.ConvertAnyTime(coalesce(BH.CarryFwd,0)) as CarryFwd2,
	   
	   
	   Case 
	   
			when AbsencesRates.UsedSeconds2 <> AbsencesRates.UsedSeconds then 'Banked booked but not Banked Rate'
			when lead(((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )))over(partition by E.empref order by E.Empref,BH.periodstart desc) <> coalesce(BH.hrsbrghtfwd,0)
				and lead(((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )))over(partition by E.empref order by E.Empref,BH.periodstart desc) = coalesce(BH.Adjustment,0)
				then 'BrtFwd Fail & Adjust = ' + dbo.ConvertAnyTime(BH.Adjustment) 
			when lead(((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )))over(partition by E.empref order by E.Empref,BH.periodstart desc) <> coalesce(BH.hrsbrghtfwd,0)
				then Coalesce(NULLIF(lead(dbo.ConvertAnyTime(((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) ))))over(partition by E.empref order by E.Empref,BH.periodstart desc),'0:00'),'')+
				' BrtFwd not transferred'
			when BankedRates.EarnedSeconds <> BH.Banked then 'Banked Differ'
			when coalesce(AbsencesRates.UsedSeconds,0) <> coalesce(BH.absbooked,0) then dbo.ConvertAnyTime(coalesce(BH.absbooked,0))+' Booked DB but Calc = '+ coalesce(AbsencesRates.USedBanked,'0')
			when CONVERT(VARCHAR(8), BH.PeriodStart, 3) not in(BankStart.periodstart) then Cast (BH.BHRef as Varchar(10)) + ' Bank Ref Start ='+ CONVERT(VARCHAR(8), BH.PeriodStart, 3)
			when CONVERT(VARCHAR(8), PB2.PeriodStart, 3) not in(PaidStart.periodstart) then Cast (PB2.PBRef as Varchar(10)) + ' Paid Ref Start ='+ CONVERT(VARCHAR(8), PB2.PeriodStart, 3)
			when PB2.PaidDate Not Between  PB2.PeriodStart  and PB2.PeriodEnd then 'Paid outside Start/End' 
													
		else ''
		END as Warnings,
		--Case 
	 --  when PB2.rateno= 0 then 'Unpaid' 
	 --  end as Warning2,
	 --lead(BH.periodstart,2) over(partition by E.empref order by E.Empref,BH.periodstart desc) as test,
	   --'Calc' as MyCalc,
		datepart(year,BH.PeriodStart) as [Year],
		Coalesce(NULLIF(lead(dbo.ConvertAnyTime(((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) 
				-(coalesce(AbsencesRates.UsedSeconds,0) ))))over(partition by E.empref order by E.Empref,BH.periodstart desc),'0:00'),'') as PrevBal,

		dbo.ConvertAnyTime(
		Case
			--when lead(((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )))over(partition by E.empref order by E.Empref,BH.periodstart desc) = coalesce(BH.Adjustment,0)
			--then Lead(coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0)))over(partition by E.empref order by E.Empref,BH.periodstart desc)
			when year(BH.periodstart)= @minyear
			then coalesce(BH.hrsbrghtfwd,0)
			
			when lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0)))over(partition by E.empref order by E.Empref,BH.periodstart desc) = 0 and
			BH.hrsbrghtfwd = 0  then 0

			when lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0)))over(partition by E.empref order by E.Empref,BH.periodstart desc) = 0 and
				lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0)),2)over(partition by E.empref order by E.Empref,BH.periodstart desc) <> 0
				then 
				lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0))-(coalesce(AbsencesRates.UsedSeconds,0)) ,2)over(partition by E.empref order by E.Empref,BH.periodstart desc)
			when BH.hrsbrghtfwd = 0 and
					lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0)))over(partition by E.empref order by E.Empref,BH.periodstart desc) <> 0 and
				lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0)),2)over(partition by E.empref order by E.Empref,BH.periodstart desc) <> 0
				then 
				lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0)))over(partition by E.empref order by E.Empref,BH.periodstart desc)+
				lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0))-(coalesce(AbsencesRates.UsedSeconds,0)) ,2)over(partition by E.empref order by E.Empref,BH.periodstart desc)

			when 
				lead(BH.hrsbrghtfwd) over(partition by E.empref order by E.Empref,BH.periodstart desc) = 0 
				and lead(BankedRates.EarnedSeconds) over(partition by E.empref order by E.Empref,BH.periodstart desc) = 0
				and lead(BH.adjustment) over(partition by E.empref order by E.Empref,BH.periodstart desc)= 0
				and lead(HT.HoursTransfered) over(partition by E.empref order by E.Empref,BH.periodstart desc) =0 
				and lead(AbsencesRates.UsedSeconds) over(partition by E.empref order by E.Empref,BH.periodstart desc) =0 
				and
				lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0)),2)over(partition by E.empref order by E.Empref,BH.periodstart desc) <> 0
				then 
				lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0))-(coalesce(AbsencesRates.UsedSeconds,0)) ,2)over(partition by E.empref order by E.Empref,BH.periodstart desc)


			when lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0)))over(partition by E.empref order by E.Empref,BH.periodstart desc) = coalesce(BH.Adjustment,0)
				then 
				lead(((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) 
				-(coalesce(AbsencesRates.UsedSeconds,0)))
				)over(partition by E.empref order by E.Empref,BH.periodstart desc)
			else
				lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0)))over(partition by E.empref order by E.Empref,BH.periodstart desc)
			END)
		
		 as PrevBal2,


		coalesce(NULLIF(dbo.ConvertAnyTime(BH.hrsbrghtfwd),'0:00'),'') as BrtFwd3,
		coalesce(BankedRates.EarnedBanked1,'') Banked3,
		coalesce(NULLIF(dbo.ConvertAnyTime(coalesce(BankedRates.Earnedseconds,0)+coalesce(BH.hrsbrghtfwd,0)),'0:00'),'') [Bank+BrFwd],
		coalesce(NULLIF(dbo.ConvertAnyTime(coalesce(BankedRates.Earnedseconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.Adjustment,0)),'0:00'),'') [Bank+BrFwd+Adj],
		
		coalesce(NULLIF(dbo.ConvertAnyTime(coalesce(BankedRates.Earnedseconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.Adjustment,0)+coalesce(HT.HoursTransfered,0)),'0:00'),'') [Bank+BrFwd+Adj+Trans],
		'-' as [-],
		coalesce(NULLIF(dbo.ConvertAnyTime(Pbanked.PaidBanked),'0:00'),'') as PaidBank,
				--sum(coalesce(PB2.paidtime,0)) as PaidBanked2,
		
		coalesce(NULLIF(AbsencesRates.USedBanked,'0:00'),'') as BookBankALL,
		coalesce(USedBanked1,'') as BankOnly,
		'=' as [=],
		coalesce(NULLIF(dbo.ConvertAnyTime((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )),'0:00'),'') as BalanceALL,
		coalesce(NULLIF(dbo.ConvertAnyTime(dbo.fnEmpBankedHoursBal(E.empref,BH.periodstart)),'0:00'),'') as BalanceFN,
		coalesce(NULLIF(dbo.ConvertAnyTime((coalesce(BankedRates.EarnedSeconds,0)+
			lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) ))over(partition by E.empref order by E.Empref,BH.periodstart desc)
			+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )),'0:00'),'') as BalWithPrevBal,
		Case
			when lead(((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )))over(partition by E.empref order by E.Empref,BH.periodstart desc) = coalesce(BH.Adjustment,0)
			then coalesce(NULLIF(dbo.ConvertAnyTime((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )),'0:00'),'')
			--when 
			else
			coalesce(NULLIF(dbo.ConvertAnyTime((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)+
				lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) ))over(partition by E.empref order by E.Empref,BH.periodstart desc))
				 -(coalesce(AbsencesRates.UsedSeconds,0) )),'0:00'),'')
			
			END as BalWithPrevBal2,
		Case
			when lead(((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )))over(partition by E.empref order by E.Empref,BH.periodstart desc) = coalesce(BH.Adjustment,0)
			then coalesce(NULLIF(dbo.ConvertAnyTime((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )),'0:00'),'')
			when year(BH.periodstart)= @minyear
			then coalesce(NULLIF(dbo.ConvertAnyTime((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )),'0:00'),'')
			else
			coalesce(NULLIF(dbo.ConvertAnyTime((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0))--,'0:00'),'')
				 + lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) ))over(partition by E.empref order by E.Empref,BH.periodstart desc)
				 - coalesce(AbsencesRates.UsedSeconds,0)),'0:00'),'')
			
			END as BalWithPrevBal3,
		
		Case
			when lead(((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )))over(partition by E.empref order by E.Empref,BH.periodstart desc) = coalesce(BH.Adjustment,0)
			then coalesce(NULLIF(dbo.ConvertAnyTime((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )),'0:00'),'')
			when year(BH.periodstart)= @minyear
			then coalesce(NULLIF(dbo.ConvertAnyTime((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )),'0:00'),'')
			else
			coalesce(NULLIF(dbo.ConvertAnyTime(
				Lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)))over(partition by E.empref order by E.Empref,BH.periodstart desc)
				 + lead((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) ),2)over(partition by E.empref order by E.Empref,BH.periodstart desc)
				 - Lead(coalesce(AbsencesRates.UsedSeconds,0))over(partition by E.empref order by E.Empref,BH.periodstart desc)),'0:00'),'')
			
			END as BalWithPrevBal4,
	   --DB values--
	   --BH.hrsbrghtfwd BrtFwd,
    --   BH.Banked,
	   --BH.Adjustment Adjust,
    --   BH.absbooked ToilBook1,
	   --BH.CarryFwd,

	   'Calc' as [Replace],
	   lead(((coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0)) -(coalesce(AbsencesRates.UsedSeconds,0) )))over(partition by E.empref order by E.Empref,BH.periodstart desc) PreBalSec,
	   coalesce(BankedRates.EarnedSeconds,0)+coalesce(BH.hrsbrghtfwd,0)+coalesce(BH.adjustment,0)+ coalesce(HT.HoursTransfered,0) -(coalesce(AbsencesRates.UsedSeconds,0) )	as BalSec,
	   coalesce(BankedRates.EarnedSeconds,0) as BankEarned,
	   coalesce(AbsencesRates.UsedSeconds,0) as BankUsed,
	   BH.BHRef

into #TempBankedHours
FROM BankedHours BH
JOIN empdetails E ON E.empref = BH.empref

JOIN absences A ON A.empref = BH.empref--and a.absdate between BH.PeriodStart and BH.PeriodEnd
Left Join PaidBanked PB2 on PB2.empref = BH.empref and Year(PB2.PeriodEnd) =  year(BH.PeriodEnd)
outer apply (select CONVERT(varchar, DATEADD(ms, SUM(CR.Banked) * 1000, 0), 114) AS EarnedBanked,
					dbo.ConvertAnyTime(SUM(coalesce(CR.Banked,0))) AS EarnedBanked1,
					SUM(CR.Banked) as EarnedSeconds			 
						from clockingrates CR
							where CR.empref = BH.empref and CR.clockdate between BH.PeriodStart and BH.PeriodEnd and CR.banked >0) as BankedRates

Outer Apply (select CONVERT(varchar, DATEADD(ms, SUM(A.banked) * 1000, 0), 114) AS USedBanked2,
					dbo.ConvertAnyTime(SUM(coalesce(NULLIF(A.Banked,-2),0))) AS USedBanked1,
					dbo.ConvertAnyTime(SUM(coalesce(NULLIF(A.rate1,-2),0)
					+coalesce(NULLIF(A.rate2,-2),0)+coalesce(NULLIF(A.rate3,-2),0)+coalesce(NULLIF(A.rate4,-2),0)+coalesce(NULLIF(A.rate5,-2),0)+coalesce(NULLIF(A.rate6,-2),0)+coalesce(NULLIF(A.rate7,-2),0)
					+coalesce(NULLIF(A.rate8,-2),0)+coalesce(NULLIF(A.rate9,-2),0)+coalesce(NULLIF(A.rate10,-2),0)+coalesce(NULLIF(A.rate11,-2),0)+coalesce(NULLIF(A.toil,-2),0)+coalesce(NULLIF(A.Banked,-2),0))) 
					AS USedBanked,

					SUM(coalesce(A.Banked,0)) as UsedSeconds2,
					SUM(coalesce(NULLIF(A.rate1,-2),0)
					+coalesce(NULLIF(A.rate2,-2),0)+coalesce(NULLIF(A.rate3,-2),0)+coalesce(NULLIF(A.rate4,-2),0)+coalesce(NULLIF(A.rate5,-2),0)+coalesce(NULLIF(A.rate6,-2),0)+coalesce(NULLIF(A.rate7,-2),0)
					+coalesce(NULLIF(A.rate8,-2),0)+coalesce(NULLIF(A.rate9,-2),0)+coalesce(NULLIF(A.rate10,-2),0)
					+coalesce(NULLIF(A.rate11,-2),0)+coalesce(NULLIF(A.toil,-2),0)+coalesce(NULLIF(A.Banked,-2),0))
					as UsedSeconds
				from absences A
					where A.empref = BH.empref and A.absdate between BH.PeriodStart and BH.PeriodEnd AND coderef = 19) as AbsencesRates
				
outer apply (select sum(coalesce(H.amount,0)) as HoursTransfered from HoursTransfer H where H.empref=BH.Empref and H.PeriodStart between  BH.PeriodStart  and BH.PeriodEnd) HT
outer apply (select sum(coalesce(PB.paidtime,0)) as PaidBanked from PaidBanked PB where PB.empref=BH.Empref and PB.PaidDate between  BH.PeriodStart  and BH.PeriodEnd) PBanked
outer apply (select CONVERT(VARCHAR(8), Min(periodstart), 3) as periodstart,year(periodstart) as Year from PaidBanked where year(periodstart) = Year(BH.periodstart)
group by year(periodstart))PaidStart
outer apply (select CONVERT(VARCHAR(8), Min(periodstart), 3) as periodstart,year(periodstart) as Year from BankedHours where year(periodstart) = Year(BH.periodstart)
group by year(periodstart))BankStart
--WHERE 
--E.payrollno in (7590,1013,1043,1005) and 
--year(BH.periodstart) IN( @year,@year2)
GROUP BY E.forenames,
		BH.PeriodStart,
         BH.periodEnd,
         BH.Banked,
         BH.absbooked,
         BH.CarryFwd,
         BH.hrsbrghtfwd,
         BH.hrsbrghtfwd,
		 BankedRates.EarnedBanked1,
		 AbsencesRates.USedBanked,
		 BankedRates.EarnedSeconds,AbsencesRates.UsedSeconds,
		 HT.HoursTransfered,
		 E.empref,
		 E.payrollno,
		 BH.adjustment,
		 BH.BHRef,
		 Pbanked.PaidBanked,
		PB2.rateno,
		PB2.PaidDate,
		PB2.PeriodStart,
		PB2.PeriodEnd,
		PB2.PBref,
		PB2.empref,
		 AbsencesRates.UsedSeconds2,
		 USedBanked1,
		 BankStart.periodstart,
		 PaidStart.periodstart

--ORDER BY e.surname;

select distinct * from #TempBankedHours
--where Warnings > ''
ORDER BY payroll,forenames,Year desc;

--Update BH
--Set BH.absbooked = TP.BankUsed
--from BankedHours BH
--Join #TempBankedHours TP on TP.BHRef = BH.BHRef
--where TP.BHRef = BH.BHRef



--Update BH
--Set BH.hrsbrghtfwd = TP.PreBalSec
--from BankedHours BH
--Join #TempBankedHours TP on TP.BHRef = BH.BHRef
--where TP.PreBalSec IS NOT NULL and TP.BHRef = BH.BHRef


GO

drop function [dbo].[ConvertAnyTime]

-- This will give same start date as the Paid date so can be found

--update PaidBanked
--set PeriodStart = dateadd(YEAR,datediff(YEAR,PeriodStart,paiddate),PeriodStart),
--PeriodEnd = dateadd(YEAR,datediff(YEAR,PeriodEnd,paiddate),PeriodEnd) 
--where 
--PaidDate Not Between PeriodStart and PeriodEnd 
