Docker Web Shell ʵ��ԭ��
===
#˵��
Docker Web Shellʵ�ִ�Web�����������SSH�ķ�ʽ��¼������Docker������

#����չʾ
�������б���棬ѡ�������������Զ�̡�������Docker����Web Shell���棺

Docker Web Shell�ṩ���ƴ�ͳSSH�ն˵��û����顣

#ʵ��ԭ��
##����
��Ҫ��ϵͳ���������Web�������Docker Controller��Docker Daemon��Docker������
+ Web��������������֡�
+ Docker Controller��Docker����Ӧ�õĿ������ģ���Ϊ������������Ϣ��ת����
+ Docker Daemon�ṩHTTP API�ӿڸ��ⲿϵͳ�����Է��������ڲ�����

##��·����
Web���������JS�ű���ͨ��Web Socket��Docker Controller����ͨ����·��
Docker Controllerͨ��Docker HTTP API��Docker Daemon����ͨ����·������ʹ�õ�Docker HTTP API��Exec Create��Exec Start��Exec Resize�ӿڣ���ͨ��hijacking����������Exec Start���ص�����������Docker Controller��Docker Daemon֮��Ľ������ݡ�

##����
��·�������û��Ϳ�����Web����������ַ���Docker����������
���û�����������м��̲���ʱ����������ͨ��Web Socketͨ������Docker Controller��Docker Controllerͨ��HTTP API�ӿ�͸����Docker Daemon��Docker Daemon�ٴ��ݸ�������
�����յ����ݺ��ٸ������ݴ���ĺ��壬����������ʱ��ͨ·������Ӧ�����ݻ��Ե�������ϡ�
ͨ�����ַ�ʽ��������˽����Ĳ�����

