sanple user data script which is triggered from tf_ami_deployer
#!/bin/bash
hostname "jboss-standalone-custcoms.dev01.aws.allegiant.com"
echo "fs-0a76ae2575c244f59 /opt/jboss_nfs efs _netdev,noresvport,tls,accesspoint=fsap-0493aa2d3d9b2fa2d 0 0" >>/etc/fstab
echo "fs-0a76ae2575c244f59 /mnt/filestore efs _netdev,noresvport,tls,accesspoint=fsap-01ab56b49532f7cfa 0 0" >> /etc/fstab
echo "fs-0a76ae2575c244f59 /opt/g4-operations/cassPhotos efs _netdev,noresvport,tls,accesspoint=fsap-06d11f68dc34ef446 0 0" >> /etc/fstab
echo "fs-0a76ae2575c244f59 /data/content/publication-manager efs _netdev,noresvport,tls,accesspoint=fsap-06c70a45f9e7f1284 0 0" >> /etc/fstab
mkdir -p /opt/jboss_nfs
mkdir -p /mnt/filestore
mkdir -p /opt/g4-operations/cassPhotos
mkdir -p /data/content/publication-manager
mount -a
#/usr/local/aws-cli/v2/2.2.20/bin/aws ssm get-parameter --name vault_lowers --with-decryption --region us-west-2 --output text --query Parameter.Value > vaultpass
#/usr/bin/ansible-playbook eap6sa_custcoms --skip-tags consul --extra-vars 'jboss_start=False g4_allow_nexus=False' --vault-password-file vaultpass --tags configuration
#rm -f vaultpass
echo "127.0.0.1 localhost $(hostname) $(hostname -s)" > /etc/hosts
echo -e '[jboss_standalone]\n127.0.0.1 ansible_connection=local working_environment=awsdev01'  > /etc/ansible/hosts
echo -e '[jbs002_custcoms]\n127.0.0.1 ansible_connection=local working_environment=awsdev01' >> /etc/ansible/hosts
echo -e '' >> /etc/ansible/hosts
echo -e '#! /bin/bash\n/usr/bin/echo $G4_VAULT_PASS' > /root/env-vault.sh
chmod 700 /root/env-vault.sh
export G4_VAULT_PASS=$(/usr/local/bin/aws ssm get-parameter --name /aws/reference/secretsmanager/devops/vault_lowers --with-decryption --region us-west-2 --output text --query Parameter.Value)
export ANSIBLE_VAULT_PASSWORD_FILE=/root/env-vault.sh
export is_live=$(/usr/local/bin/aws ssm get-parameter --name /env_info/is_env_live --with-decryption --region us-west-2 --output text --query Parameter.Value)
mkdir -p /root/repo/sdlc_deploy_s3
cd /root/repo/sdlc_deploy_s3
/usr/local/bin/aws s3 cp s3://878031627345-us-west-2-deployment-archive/jboss-standalone-custcoms/sdlc_deploy.tar . --endpoint-url https://bucket.vpce-052dffaae324ab23c-1agcyw3v.s3.us-west-2.vpce.amazonaws.com
/usr/local/bin/aws s3api head-object --bucket 878031627345-us-west-2-deployment-archive --key jboss-standalone-custcoms/sdlc_deploy.tar --endpoint-url https://bucket.vpce-052dffaae324ab23c-1agcyw3v.s3.us-west-2.vpce.amazonaws.com
tar xvzf sdlc_deploy.tar --strip-components=1
ansible-playbook "eap6sa_custcoms.yml" --skip-tags sumo,consul,datadog --extra-vars 'jboss_start=False' -e is_live=${is_live} > /var/log/ansible.log
if [ -s /var/log/ansible.log ]; then echo "/var/log/ansible.log file is not empty" ;ansible_out=$(cat /var/log/ansible.log |grep failed=|grep -v failed=0);if [ -n "$ansible_out" ]; then echo "Ansible run encountered failures, check ansible logs in /var/log/ansible.log"; systemctl stop jboss-custcoms; exit 1; else echo "Ansible run completed successfully (no failures detected)"; fi; else "/var/log/ansible.log file is empty" ; systemctl stop jboss-custcoms; exit 1;fi

ifconfig |grep "inet " |grep broadcast|awk '{print $2"  "}' |tr -d '\n' >> /etc/hosts
echo "$(hostname) $(hostname -s)" >> /etc/hosts
systemctl enable collector
systemctl start collector
sleep 30
rm -rf /etc/sumo.conf
/opt/CrowdStrike/falconctl -s --cid=`/usr/local/aws-cli/v2/2.2.20/bin/aws ssm get-parameter --name crowdstrikeCID --with-decryption --region us-west-2 --output text --query Parameter.Value`
systemctl start falcon-sensor

export SUB_ENV=$(/usr/local/bin/aws ssm get-parameter --name /aft/account-request/custom-fields/account/sub_env --with-decryption --region us-west-2 --output text --query Parameter.Value)
export is_live=$(/usr/local/bin/aws ssm get-parameter --name /env_info/is_env_live --with-decryption --region us-west-2 --output text --query Parameter.Value)
if [ dev01 == "prd01" ]; then serveralias="-dev01.allegiantair.com"; else serveralias=".dev01.aws.allegiantair.com"; fi
if [[ dev01 == "prd01" && ${is_live} == "true" ]]; then serveralias_conf=".allegiantair.com"; else serveralias_conf=${serveralias}; fi
sed -i "s/www.infra.aws.allegiant.com/www${serveralias_conf}/g" /etc/jbossas/standalone/sa-custcoms.xml
sed -i "s+.*<property name=\"environment.name.*+        <property name=\"environment.name\" value=\"dev01.aws.allegiantair.com\"/>+" /etc/jbossas/standalone/sa-custcoms.xml
sed -i "s/infra.aws.allegiant.com/$SUB_ENV.aws.allegiant.com/g" /etc/jbossas/standalone/sa-custcoms.xml
if [ `/usr/local/aws-cli/v2/2.2.20/bin/aws ssm get-parameter --name /env_info/is_env_live --with-decryption --region us-east-2 --output text --query Parameter.Value` != "false" ]; then
  sed -i "s+.*cheetah.password.*+        <property name=\"cheetah.password\" value=\"`/usr/local/aws-cli/v2/2.2.20/bin/aws ssm get-parameter --name /jbossSA/stg/cheetah.password --with-decryption --region us-east-2 --output text --query Parameter.Value`\"/>+" /etc/jbossas/standalone/sa-custcoms.xml
  sed -i "s+.*cheetah.username.*+        <property name=\"cheetah.username\" value=\"`/usr/local/aws-cli/v2/2.2.20/bin/aws ssm get-parameter --name /jbossSA/stg/cheetah.username --with-decryption --region us-east-2 --output text --query Parameter.Value`\"/>+" /etc/jbossas/standalone/sa-custcoms.xml
  sed -i 's+.*emailValidator.filter.inclusive.*+        <property name="emailValidator.filter.inclusive" value="true"/>+' /etc/jbossas/standalone/sa-custcoms.xml
  sed -i 's/.*emailValidator.filter.regex.*/        <property name="emailValidator.filter.regex" value="^(?i)([^@]+@?(tridentsqa.com|allegiantair.com|kayasoftware.com|mailinator.com))|(ccmod.validation@gmail.com)$|(adinnerod@gmail.com)$"\/>/' /etc/jbossas/standalone/sa-custcoms.xml
  sed -i 's/.*environment.name.*/        <property name="environment.name" value="stg"\/>/' /etc/jbossas/standalone/sa-custcoms.xml
  sed -i s/resweb_/resweb_stage_/ /etc/jbossas/standalone/sa-custcoms.xml
  sed -i s/otares_/otares_stage_/ /etc/jbossas/standalone/sa-custcoms.xml
  sed -i s/reaccom_/reaccom_stage_/ /etc/jbossas/standalone/sa-custcoms.xml
  sed -i s/myaccount_/myaccount_stage_/ /etc/jbossas/standalone/sa-custcoms.xml
  sed -i s/fclub_verification/fclub_stage_verification/ /etc/jbossas/standalone/sa-custcoms.xml
  sed -i s/rewards_accountlinked/rewards_stage_accountlinked/ /etc/jbossas/standalone/sa-custcoms.xml
fi
export dynatrace_tenant=$(/usr/local/bin/aws ssm get-parameter --name /aws/reference/secretsmanager/dynatrace/tenantID --with-decryption --region us-west-2 --output text --query Parameter.Value)
export dynatrace_tenant_token=$(/usr/local/bin/aws ssm get-parameter --name /aws/reference/secretsmanager/dynatrace/tenantToken --with-decryption --region us-west-2 --output text --query Parameter.Value)
if [ `/usr/local/aws-cli/v2/2.2.20/bin/aws ssm get-parameter --name /env_info/monitoring_mode  --with-decryption --region us-west-2 --output text --query Parameter.Value` == "ON" ]; then /bin/sh /root/Dynatrace-OneAgent-Linux-1.249.198.sh --set-infra-only=false --set-app-log-content-access=true --set-host-group=env_dev01 --set-host-tag=custcoms --set-host-property=env=dev01 --set-host-tag=env=dev01 --set-tenant=${dynatrace_tenant} --set-tenant-token=${dynatrace_tenant_token} --set-server=https://${dynatrace_tenant}.live.dynatrace.com/; fi
/etc/init.d/jboss-custcoms start
sleep 10
service_stat=$(systemctl status jboss-custcoms|grep "active (running)")
if [ -n "$service_stat" ]; then echo "Service status is OK" ;else echo "Service status is Not OK";exit 1; fi
if [ true == true ]; then tail -n 100 $log_file |grep -iE 'error|fail'; if [ $? -eq 0 ]; then echo "Errors or failure detected in the logs"; systemctl stop jboss-custcoms;exit 1; else  echo " No Errors or failure detected in the logs";fi; else echo "application log check is false";fi
