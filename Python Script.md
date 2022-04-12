# HEAD-Competition--Hamilton-Municipal-Parking-Lots-and-User-Transaction-Analysis
#The following code calculates the number of chargeable minutes per transaction
import numpy as np
from business_duration import businessDuration
import pandas as pd
from datetime import time,datetime
import holidays as pyholidays
from datetime import timedelta


df = pd.read_excel('MasterHEADFile.xlsx')
df['Exit'] = pd.to_datetime(df['Exit Day']+ ' ' +df['Exit Time'])
df['Entry'] = pd.to_datetime(df['Entry Date']+ ' ' +df['Entry Time'].astype(str))
#these two lines make datetime stamps, but we can get the timestamps from the original copies
df['Close'] = pd.to_datetime(df['Exit Day']+ ' ' +df['Closing Hour of Operation'].astype(str))
df['Open'] = pd.to_datetime(df['Entry Date']+ ' ' +df['Opening Hour of Operation'].astype(str))


closes_am = df['Closing Hour of Operation'] < df['Opening Hour of Operation']
#this is the 2:00AM anomally that I could't get rid of, it just kept creeping back
df.loc[closes_am & (df['Exit'] < df['Entry']), 'Exit'] = df['Exit'] + timedelta(days = 1, hours = 0, minutes = 0, seconds = 0)
#add a day to the Exit Time if they park after 12AM, the clause is if the Exit Time < Entry Time, which is what happens to
#the parking lots with a 2AM close
df.loc[closes_am, 'Entry'] = df['Entry'] - timedelta(days = 0, hours = 3, minutes = 0, seconds = 0)
df.loc[closes_am, 'Exit'] = df['Exit'] - timedelta(days = 0, hours = 3, minutes = 0, seconds = 0)
#shift back all the entry and exit hours by 3 hours, to shift back this day from 2AM-->11PM and from 9AM-->6AM or 8AM-->5AM.
#It should be noted that the opening hour and the closing hour need to shift back as well that is done in the next few lines

df['close_hour'] = df.Close.dt.hour
df['open_hour'] = df.Open.dt.hour
df.loc[closes_am, 'close_hour'] = 23
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

df['chargeable_minutes'] = df.apply(lambda x: businessDuration(startdate = x['Entry'],enddate = x['Exit'], \
                                                               starttime =time(x['open_hour'], 0,0),\
                                                               endtime= time(x['close_hour'], 0,0), \
                                                               holidaylist= holidaylist,\
                                                               weekendlist = [x['Weekends']],\
                                                               unit='min'), axis =1)
#We calculate the chargeable minutes

df['HoursOpen']=(df['close_hour']-df['open_hour'])*60
#a full day is no longer 1440 mins, its the number of hours open *60
df['NumDays']=np.floor(df['chargeable_minutes']/df['HoursOpen'])
df['RemainingMins']=np.ceil(np.mod(df['chargeable_minutes'], df['HoursOpen']))

condition1= (df['chargeable_minutes']) == 0
# then 0
condition2 = (df['chargeable_minutes'] >= df['Minutes for Daily Rate']) & (df['RemainingMins'] == 0)
#then daily rate*df['Numdays']
condition3 = (df['chargeable_minutes'] >= df['Minutes for Daily Rate']) & (df['RemainingMins'] >= df['Minutes for Daily Rate']) 
#then rate*df['Numdays'] + daily rate
condition4 = (df['chargeable_minutes'] >= df['Minutes for Daily Rate']) & (df['RemainingMins'] < df['Minutes for Daily Rate']) 
#then rate*df['Numdays'] + df['RemainingMins']* Rate/minmin
condition5= (df['chargeable_minutes'])<(df['Minutes for Daily Rate'])
#then chargemins*Rate/minmin
condition6= (df['All Day Rate']=0)
#then chargemins*Rate/minmin

df.loc[condition1, 'PlotFee'] =0
df.loc[condition2, 'PlotFee'] = df['NumDays']*df['All Day Rate']
df.loc[condition3, 'PlotFee'] = (df['NumDays']+1)*df['All Day Rate']
df.loc[condition4, 'PlotFee'] = df['NumDays']*df['All Day Rate'] + np.ceil(df['RemainingMins']/df['Min Minutes'])*df['Rate/min min']
df.loc[condition5, 'PlotFee'] = np.ceil(df['chargeable_minutes']/df['Min Minutes'])*df['Rate/min min']
df.loc[condition6, 'PlotFee'] = np.ceil(df['chargeable_minutes']/df['Min Minutes'])*df['Rate/min min']
#calculate the fees

df.loc[df['close_hour']==23, ['Exit','Entry','Close','PlotFee','chargeable_minutes','Rate/min min',\
                           'All Day Rate','NumDays','RemainingMins','Min Minutes','Minutes for Daily Rate']]
#this line is for sanity checks, the first input can be replaced with any logical expression to test the data and make sure it is
#accurate and try to debug, if necessary
