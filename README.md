Kempy's Process Monitor Alpha release 0.1

The purpose of this driver is to record system activity on the host currently to a logfile at c:\windows\kpm-log.txt
This driver only logs created process details at this time in the following format. 

Demo video can be found here https://youtu.be/Xad5nUmKr1w

kpm - 2015-12-06T15:10 DESKTOP-9SCP4K5 Parent[PID=4948,UP:UT=4948:2532,Image=\Device\HarddiskVolume2\Windows\System32\cmd.exe] Child[PROC=1014941568,PID=4416,CLI=hostname,Image=\??\C:\Windows\system32\HOSTNAME.EXE] 

This is currently useful for call chain monitoring or seeing what is running accross your network at this time.
I will be adding more features as time permits.

I use NxLog to monitor this logfile and ship events to a ELK (Elastic, Logstash and Kibana) server.
I am using version nxlog-ce-2.8.1248.msi obtainable from https://nxlog.org/products/nxlog-community-edition/download
You should edit the config normally at c:\program files(x86)\nxlog\conf\nxlog.conf
my config is as follows;



				## This is a sample configuration file. See the nxlog reference manual about the
				## configuration options. It should be installed locally and is also available
				## online at http://nxlog.org/nxlog-docs/en/nxlog-reference-manual.html

				## Please set the ROOT to the folder your nxlog was installed into,
				## otherwise it will not start.

				#define ROOT C:\Program Files\nxlog
				define ROOT C:\Program Files (x86)\nxlog

				Moduledir %ROOT%\modules
				CacheDir %ROOT%\data
				Pidfile %ROOT%\data\nxlog.pid
				SpoolDir %ROOT%\data
				LogFile %ROOT%\data\nxlog.log

				<Input in>
					Module      im_msvistalog

				# For windows 2003 and earlier use the following:
				#   Module      im_mseventlog
				</Input>

				<Input in1>
					Module im_file
					File 'c:\windows\kpm-log.txt'
					SavePos TRUE
					ReadFromLast TRUE
					PollInterval 30
					Exec $Message = $raw_event; $SyslogFacilityValue = 22;
				</Input>

				<Output out>
					Module      om_tcp
					Host        192.168.56.10
					Port        5514
				</Output>

				<Route 1>
					Path        in, in1 => out
				</Route>
				############## end of config file #################

you should start the service (via services.msc) or reboot.

you can obtain an ELK server from the OSSEC project. my ELK server was built on Centos from scratch and I don't have the details of what I did.
grab the virtual appliance (ossec-vm-2.8.2.ova) from http://ossec.github.io/downloads.html

I used virtualBox 5.0.2 with the extensions installed for my ELK server and target test systems which use the default host only network of 192.168.56.x

unfortunately you will need to manually install the driver package.

Currently you will need to switch your desktop into test mode. runas administrator cmd.exe then enter following command;
 
		Bcdedit.exe -set TESTSIGNING ON
		
	And/or
		BCDEDIT /set nointegritychecks ON

Next we install my test certificate and then through device manager add a new device.

select cert > right click > install certificate.
* select 'Local Machine' > Next, Yes to consent, * Place all cert in following store > browse, select [Truste root cert auth OR Trusted publishers] > Next
* run above step again selecting the other store yet to be installed to.

* in device manager, right click computername > add legacy hardware > next > select 'install hardware manually' > next > showall device, next > have disk, browse to driver package and select .inf file > ok > select the driver, next finish.

now check the local logfile has events. open cmd prompt and ping or hostname. check these events show in log file at times you ran them.

Tweak your centralised log server to index events(event format may change in future)

I used this site https://grokdebug.herokuapp.com/ to build the grok format below;

		kpm - %{TIMESTAMP_ISO8601:HostTimeStamp} %{HOSTNAME:Hostname} Parent\[PID=%{INT:ParentPID},UP:UT=%{INT:pProcessID}:%{INT:pThreadID},Image=+%{GREEDYDATA:pImage}\] Child\[PROC=%{INT:Process},PID=%{INT:ChildPID},CLI=+%{GREEDYDATA:CLI},Image=+%{GREEDYDATA:cImage}\]

Logstash sample 
nano /etc/logstash/conf.d/10-syslog.conf

		filter {
		  if [type] == "syslog" {
			grok {
			  match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
			  add_field => [ "received_at", "%{@timestamp}" ]
			  add_field => [ "received_from", "%{host}" ]
			}
			syslog_pri { }
			date {
			  match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
			}
			grok {
			match => { "message" => "kpm - %{TIMESTAMP_ISO8601:HostTimeStamp} %{HOSTNAME:Hostname} Parent\[PID=%{INT:ParentPID},UP:UT=%{INT:pProcessID}:%{INT:pThreadID},Image=+%{GREEDYDATA:pImage}\] Child\[PROC=%{INT:Process},PID=%{INT:ChildPID},CLI=+%{GREEDYDATA:CLI},Image=+%{GREEDYDATA:cImage}\]" }
			add_tag => "kpm grokked"
			}
		  }
		}



nano /etc/logstash/conf.d/20-central.conf

		input
		{
		#       redis
		#       {
		#               host => "192.168.0.1"
		#               data_type => "list"
		#               type => "redis-input"
		#               key => "logstash"
		#       }
				syslog
				{
						type => syslog
						port => 5514
						host => "192.168.56.10"
				}
		}
		output
		{
				stdout { }
				elasticsearch
				{
						host => localhost
				}
		}


Systems tested so far;
Windows 10

Known issues on Windows 7


Post bugs and feedback on github :)



Creative Commons License
This work is licensed under a Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International License.
http://creativecommons.org/licenses/by-nc-nd/4.0/legalcode

You are free to:

Share — copy and redistribute the material in any medium or format
The licensor cannot revoke these freedoms as long as you follow the license terms.

Under the following terms:

Attribution — You must give appropriate credit, provide a link to the license, and indicate if changes were made. You may do so in any reasonable manner, but not in any way that suggests the licensor endorses you or your use.
NonCommercial — You may not use the material for commercial purposes.
NoDerivatives — If you remix, transform, or build upon the material, you may not distribute the modified material.
No additional restrictions — You may not apply legal terms or technological measures that legally restrict others from doing anything the license permits.

Notices:

You do not have to comply with the license for elements of the material in the public domain or where your use is permitted by an applicable exception or limitation.
No warranties are given. 
