# Windows-backendƪ

���ڶ�.net����Ϥ������ֻ������Ͳ���ֻ��˵��ĿǰV3������֧����windows .net�����ˡ�</br>

## ˵��

����ĿǰV3�滹���з��Ͳ����У�����һЩ���ӵ�.NET���򣬸������ﻹ��û�ף�windows��Ҷ����ġ�</br>

## ��װ����

1.����windows2012 R2�ٷ����񣬸���һ�䣬�������װ���ҵ�virtualBox�ϣ���������������һ��Ĭ��net��ʽ��һ���Žӵ�bosh-lite�����</br>

		�Žӵ�Adapt-host-2��192.168.50.4��bosh-lite���������IPΪ192.168.50.6������Ϊ192.168.50.4
		
		Net���ùٷ�Ĭ�ϣ���֤�������ܹ���ͨ
		
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/network.JPG) <br />

2.���عٷ�ָ����ʼ���ű�����ִ�У�</br>

		https://github.com/cloudfoundry-incubator/diego-windows-msi/releases/tag/v0.694
		//power shell��ִ�У�Ҳ����cmd��ִ��
		setup.ps1
		
����Ĺ��̾������÷���ǽ��������windows���£�ǰ���Ǳ����������û�������ȷʵ��Ҫע�⣬�ڰ�װ���������Ҫ�����������**�����û�**����������һ���
.net com����ȵ�</br>

3.����windows-backend��װ����</br>

		msiexec /norestart /i c:\temp\DiegoWindowsMSI.msi ^
		ADMIN_USERNAME=Administrator ^
		ADMIN_PASSWORD=a12345678A ^
		BBS_CA_FILE=c:\temp\bbs_ca.crt ^
		BBS_CLIENT_CERT_FILE=c:\temp\bbs_client.crt ^
		BBS_CLIENT_KEY_FILE=c:\temp\bbs_client.key ^
		CONSUL_IPS=10.244.0.54 ^
		CONSUL_ENCRYPT_FILE=c:\temp\consul_encrypt.key ^
		CONSUL_CA_FILE=c:\temp\consul_ca.crt ^
		CONSUL_AGENT_CERT_FILE=c:\temp\consul_agent.crt ^
		CONSUL_AGENT_KEY_FILE=c:\temp\consul_agent.key ^
		CF_ETCD_CLUSTER=http://10.244.16.130:4001 ^
		STACK=windows2012R2 ^
		REDUNDANCY_ZONE=windows ^
		LOGGREGATOR_SHARED_SECRET=loggregator-secret ^
		EXTERNAL_IP=192.168.50.6 ^
		MACHINE_NAME=WIN-6KT73FK5PFV
		
����diego�������ͨ��consulʵ�ַ����ֵģ�����������������֤��ʱ��һ����ע�⣬����һ����BBS����������Ҫ���ܹ�ͨ��receptor��API����ʹ�ú��windows-garden�ܹ������������У�һ������LRP��TASK</br>

��������Ҫ����**external_ip**,���IP�����������ڲ����ⲿ��ͨ�ģ�����������粻�ܻ������������޷��ҵ��ʺϵ�CELL����</br>

![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/services.JPG) <br />

4.����һ������APP��ͨ�������ϴ���CF</br>

* �ȸ㶨��������������֮ǰʹ��dockerʱ�Ѿ���װ����</br>
		
		cf install-plugin -r CF-Community Diego-Beta
		cf install-plugin -r CF-Community Diego-SSH
		
* �ϴ�Ӧ��</br>
		
		git clone https://github.com/Canopy-OCTO/cip-windows-test-app
		cd cip-windows-test-app
		cf push dotnet-test-app -s windows2012R2 -b https://github.com/ryandotsmith/null-buildpack -m 1G --no-start
		cf enable-diego dotnet-test-app
		cf disable-ssh dotnet-test-app
		cf start dotnet-test-app
		
* scaleӦ��</br>

		cf scale dotnet-test-app -i 3
		
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/runningapp.JPG) <br />
		
5.����ɹ������Ե����windows�����Ͽ���һЩ��Ϊ</br>

* �ػ�����**Guard**��ÿ����һ��ʵ�����������һ��,����ԱȨ��</br>
* **IronFrame.Host** ������������C��ͷ�������û�</br>
* **launcher** �������̣���Ҫ������һЩ.net��̬�⣬���ս�����run����,��C��ͷ�������û�</br>
* **WebAppserver** Ӧ�ó�����̣���C��ͷ�������û�,������Щ���̶���ʵ���ж��پͻ���������</br>

![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/process.JPG) <br />

## �۲�����Ŀ¼�ṹ
* ��Ŀ¼</br>
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/containerdir.JPG) <br />
* �����ڲ�userĿ¼</br>
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/content.JPG) <br />
* lifecycleĿ¼</br>
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/lifecycle.JPG) <br />

���Կ���������Ϥ�Ľ��̡�</br>

## Ӧ�õĻ�������

curl http://dotnet-test-app.10.244.0.34.xip.io/env</br>

		{
		  "SystemDrive": "C:", 
		  "ProgramFiles(x86)": "C:\Program Files (x86)", 
		  "Path": "C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\", 
		  "ProgramW6432": "C:\Program Files", 
		  "PROCESSOR_IDENTIFIER": "Intel64 Family 6 Model 60 Stepping 3, GenuineIntel", 
		  "CF_STACK": "windows2012R2", 
		  "TMP": "C:\Windows\TEMP", 
		  "PROCESSOR_ARCHITECTURE": "AMD64", 
		  "CF_INSTANCE_PORT": "50460", 
		  "VCAP_SERVICES": "{}", 
		  "PROCESSOR_REVISION": "3c03", 
		  "TEMP": "C:\Windows\TEMP", 
		  "USERPROFILE": "C:\containerizer\858DE92B64DF613951\user\", 
		  "USERNAME": "SYSTEM", 
		  "SystemRoot": "C:\Windows", 
		  "PORT": "50460", 
		  "MEMORY_LIMIT": "1024m", 
		  "FP_NO_HOST_CHECK": "NO", 
		  "CF_INSTANCE_INDEX": "0", 
		  "CommonProgramFiles": "C:\Program Files\Common Files", 
		  "__COMPAT_LAYER": "ElevateCreateProcess", 
		  "ProgramData": "C:\ProgramData", 
		  "LANG": "en_US.UTF-8", 
		  "PATHEXT": ".COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC", 
		  "COMPUTERNAME": "WIN-6KT73FK5PFV", 
		  "ARGJSON": "["app","","{\"start_command\":\"tmp/lifecycle/WebAppServer.exe\",\"start_command_args\":[\".\"]}"]", 
		  "CF_INSTANCE_GUID": "2ab613f5-7c40-447c-540d-c269bb0f5e25", 
		  "ALLUSERSPROFILE": "C:\ProgramData", 
		  "CommonProgramW6432": "C:\Program Files\Common Files", 
		  "CommonProgramFiles(x86)": "C:\Program Files (x86)\Common Files", 
		  "CF_INSTANCE_IP": "192.168.50.6", 
		  "windir": "C:\Windows", 
		  "NUMBER_OF_PROCESSORS": "1", 
		  "OS": "Windows_NT", 
		  "ProgramFiles": "C:\Program Files", 
		  "ComSpec": "C:\Windows\system32\cmd.exe", 
		  "INSTANCE_GUID": "2ab613f5-7c40-447c-540d-c269bb0f5e25", 
		  "PSModulePath": "C:\Windows\system32\WindowsPowerShell\v1.0\Modules\", 
		  "INSTANCE_INDEX": "0", 
		  "CF_INSTANCE_ADDR": "192.168.50.6:50460", 
		  "PROCESSOR_LEVEL": "6", 
		  "CF_INSTANCE_PORTS": "[{"external":50460,"internal":8080}]", 
		  "VCAP_APPLICATION": "{"limits":{"mem":1024,"disk":1024,"fds":16384},"application_id":"d6a19802-d7f9-4a31-88db-82ff7ad361ee","application_version":"edec73a9-2289-48d1-8991-cec51839f6c6","application_name":"dotnet-test-app","version":"edec73a9-2289-48d1-8991-cec51839f6c6","name":"dotnet-test-app","space_name":"cloud","space_id":"9a42bc77-39d5-4175-9c93-d550e376de1a"}", 
		  "APP_POOL_ID": "AppPool50460", 
		  "PUBLIC": "C:\Users\Public", 
		  "APP_POOL_CONFIG": "C:\containerizer\858DE92B64DF613951\user\tmp\lifecycle\config\applicationHost.config"
		}
		
## һЩС����
��־�޷�д��bug</br>
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/issue.JPG) <br />

## �ܽ�
Ŀǰ���ֻ����Ϊ��ʾ�����߱����Ϸ��������˶�.net����ð������Ҳ�͵���Ϊֹ�ˡ�����Դ�룬������о������ھ�������docker�ϡ�	