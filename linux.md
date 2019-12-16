
* `systemctl start ������(�ļ���)` ��������
* `systemctl status ������` �鿴������Ϣ
* `systemctl restart ������` ��������
* `systemctl stop ������` ֹͣ����
* `systemctl enable ������` ������������
* `systemctl disable ������` ��ֹ������������
* `systemctl list-units --type=service` �鿴�����������ķ���
* `rpm -i xxx.rpm` ��װ���
* `rpm -e xxx` ж�����
* `rpm -qa | grep xxx` �鿴rpm��װ�����

#### awk

> test:test2:test3   text.txt

* `FS` ����ָ��� Ĭ�Ͽո�
* `OFS` ����ָ��� Ĭ�Ͽո�
* `RS` �����¼�ָ��� Ĭ�ϻ��з�
* `ORS` �����¼�ָ��� Ĭ�ϻ��з�
* `NF` ��ǰ��¼����ĸ���,��ÿ���ж�����
* `NR` �Ѿ������ļ�¼��
* `FILENAME` ��ǰ�ļ�������
* `awk '{print $0 }' text.txt` ��ӡһ�м�¼ Ĭ�ϻ��з�Ϊһ�м�¼ -> **test:test2:test3**
* `awk '{print NR $0} text.txt'`  NR��ʾ�Ѷ�ȡ������ ��1��ʼ,�Զ��ۼ�  -> **1test:test2:test3**
* `awk '{print $1 $3 }' text.txt` ��ʾ��ӡ��һ���͵�����Ԫ�� Ĭ���Կո���Ϊ�ָ���,��˸������ӡ
* `awk '$3="test3" {print $0}' text.txt` ����ָ���ĵ�����Ԫ�ص���ָ����test3���ӡ��һ�м�¼ `{print $0}` ���Բ���Ҫ,Ĭ�ϴ�ӡ��ǰ�м�¼
* `awk '{print $0}' text.txt > text2.txt` ������ض���text2�ļ���
* `awk -F: '{print $1}' text.txt` ��:��Ϊ�ָ��� ->**test2**
* `awk -F'[ ]' '{print $1, $3}' text.txt` �Կո���Ϊ�ָ���
* `awk -F'[,]' '{print $1, $3}' text.txt` �Զ�����Ϊ�ָ���
* `awk -v FS=':' '{print $1 $3}' text.txt` �Էֺ���Ϊ�ָ��� -> **test test3**
* `awk -v OFS="->" '{print $2, $3}' other.txt` -> **test2->test3**
* 

#### �ճ�����

* `iotop -o` ��ǰ����д���̲��������н���ID��Ϣ
* `pidstat -u -x 3` �鿴cpuʹ���ʽϸߵĽ���
* `grep -v "^#" redis.conf` ���ε�ע������Ϣ
* `echo "" > nohup.out` ���ļ����
* `ps -p [PID] -o etime` �鿴Ӧ�ó�������ʱ��
