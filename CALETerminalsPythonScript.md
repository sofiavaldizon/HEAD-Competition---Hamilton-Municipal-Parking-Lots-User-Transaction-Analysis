# HEAD-Competition---Hamilton-Municipal-Parking-Lots-User-Transaction-Analysis
#The following code calculates the fees per transaction of the CALE Terminal data
#Two different datasets were given, one for 
import numpy as np
from business_duration import businessDuration
import pandas as pd
from datetime import time,datetime
import holidays as pyholidays
from datetime import timedelta

df = pd.read_excel('CALE_Clean_08-03.xlsx')

df['Close'] = pd.to_datetime(df['End_Date'].astype(str)+ ' ' +df['Ending Hour'].astype(str))
df['Open'] = pd.to_datetime(df['Start_Date'].astype(str)+ ' ' +df['Beginning Hour'].astype(str))


closes_am = df['Ending Hour'] < df['Beginning Hour']
df.loc[closes_am & (df['End'] < df['Start']), 'End'] = df['End'] + timedelta(days = 1, hours = 0, minutes = 0, seconds = 0)

df.loc[closes_am, 'Start'] = df['Start'] - timedelta(days = 0, hours = 3, minutes = 0, seconds = 0)
df.loc[closes_am, 'End'] = df['End'] - timedelta(days = 0, hours = 3, minutes = 0, seconds = 0)
#shift back all the entry and exit hours by 3 hours, to shift back this day from 2AM-->11PM and from 9AM-->6AM or 8AM-->5AM.
#It should be noted that the opening hour and the closing hour need to shift back as well that is done in the next few lines


df['close_hour'] = df.Close.dt.hour
df['open_hour'] = df.Open.dt.hour
df.loc[closes_am, 'close_hour'] = 23  #**
#All lots that close at 2AM will now be closed at a shifted 11PM, so set the close hour for 23 hundred hours
df.loc[closes_am, 'open_hour'] = df['open_hour'] -3
#Shift back the opening hour by 3 hours


weekendcondition= df['Days of Operation']=='Monday-Saturday'
df['Weekends']=None
df.loc[weekendcondition,'Weekends']=6
#This is to take into account weekends, it automatically charges 0 chargeable minutes on Sundays but only if the row meets
#weekendcondition

holidaylist = pyholidays.Canada()
#This takes into account Canadian holidays

df['chargeable_mins'] = df.apply(lambda x: businessDuration(startdate = x['Start'],enddate = x['End'], \
                                                               starttime =time(x['open_hour'], 0,0),\
                                                               endtime= time(x['close_hour'], 0,0), \
                                                               holidaylist= holidaylist,\
                                                               weekendlist = [x['Weekends']],\
                                                               unit='min'), axis =1)
#We calculate the chargeable minutes

df['HoursOpen']=(df['close_hour']-df['open_hour'])*60
#a full day is no longer 1440 mins, its the number of hours open *60
df['NumDays']=np.floor(df['chargeable_mins']/df['HoursOpen'])
df['RemainingMins']=np.ceil(np.mod(df['chargeable_mins'], df['HoursOpen']))

condition1= (df['chargeable_mins']) == 0
# then 0
condition2 = (df['chargeable_mins'] >= df['Minutes for Daily Rate']) & (df['RemainingMins'] == 0)
#then daily rate*df['Numdays']
condition3 = (df['chargeable_mins'] >= df['Minutes for Daily Rate']) & (df['RemainingMins'] >= df['Minutes for Daily Rate']) 
#then rate*df['Numdays'] + daily rate
condition4 = (df['chargeable_mins'] >= df['Minutes for Daily Rate']) & (df['RemainingMins'] < df['Minutes for Daily Rate']) 
#then rate*df['Numdays'] + df['RemainingMins']* Rate/minmin
condition5= (df['chargeable_mins'])<(df['Minutes for Daily Rate'])
#then chargemins*Rate/minmin
condition6= (df['chargeable_mins']>0) & (df['Minutes for Daily Rate']==0)
#then chargemins*Rate/minmin

df.loc[condition1, 'PlotFee'] =0
df.loc[condition2, 'PlotFee'] = df['NumDays']*df['All Day Rate']
df.loc[condition3, 'PlotFee'] = (df['NumDays']+1)*df['All Day Rate']
df.loc[condition4, 'PlotFee'] = df['NumDays']*df['All Day Rate'] + np.ceil(df['RemainingMins']/df['Min Minutes'])*df['Min Minutes rate']
df.loc[condition5, 'PlotFee'] = np.ceil(df['chargeable_mins']/df['Min Minutes'])*df['Min Minutes rate']
df.loc[condition6, 'PlotFee'] = np.ceil(df['chargeable_mins']/df['Min Minutes'])*df['Min Minutes rate']
#calculate the fees

