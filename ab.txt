import json
import boto3
import pymysql
import threading
import logging
import requests
from pconst import const
from requests.auth import HTTPBasicAuth
from datetime import datetime, timedelta, timezone
from botocore.exceptions import ClientError

# Logging Configurations
current_date = datetime.now().strftime("%d-%m-%Y")
current_time = datetime.now().strftime("%H.%M.%S")
log_file_name = f"Logs/Discovery-Logs-{current_date}-{current_time}.log"
log_file_name = log_file_name.replace(":", "-")
logging.basicConfig(filename=log_file_name, format='%(asctime)s %(message)s')
logger = logging.getLogger()
logger.setLevel(logging.INFO)
# Logging Configurations ends here

# Secret Manager code block to fetch Database credentials
logger.info("Starting AWS Discovery Script")
logger.info("Secret Manager block starts")
secretName = "rds!db-65702254-8215-461f-a6db-fec365f37a61"   # Replace it with Secret Name of McD
regionName = "eu-central-1"       # Replace it with Secret Region of McD

session = boto3.session.Session()
client = session.client(
	service_name='secretsmanager',
	region_name=regionName
)
try:
	getSecretValueResponse = client.get_secret_value(
		SecretId=secretName
	)
	logger.info("Credential fetched")
except ClientError as e:
	logger.info(f"Error :  {str(e)}")

secret = getSecretValueResponse['SecretString']
data = json.loads(secret)
username = data['username']
password = data['password']
host = 'cmdb-dev-db.c19fhdmekbvm.eu-central-1.rds.amazonaws.com'
# Secrets Manager block ends here

# Database credentials
db_host= host
db_user= username 
db_pass= password
db_name= "AWS_Discovery"   # Replce it with the database name of McD RDS Server

# Code block to create Master session
logger.info("Creating Master Session")
sts = boto3.client('sts')
masterRole = 'arn:aws:iam::182428597156:role/MasterRole' # This ARN needs to be replace with Master Role ARN of McD 
master = sts.assume_role(
		RoleArn=masterRole,
		RoleSessionName='master_session',
		DurationSeconds=28800 
	)
credentials = master['Credentials']

accountSession = boto3.Session(
	aws_access_key_id=credentials['AccessKeyId'],
	aws_secret_access_key=credentials['SecretAccessKey'],
	aws_session_token=credentials['SessionToken'],
	region_name='us-east-1' 
)
logger.info("Master Session created")
# Code block for Master session ends here

# Constants and Global variables
const.ERROR_MESSAGE = 'Error'
const.ROLE_ERROR = 'Error Role to this account is not Attached'
const.EXECUTION_ERROR = 'AWS Cloud Ingestion Engine Error'
all_regions =  ['us-east-1', 'us-east-2', 'eu-central-1', 'eu-west-1', 'ap-southeast-1', 'ap-northeast-1']
logger.info("Fetching data from these regions : %s", all_regions)
serviceAccounts = []
accountList = []
accountsWithoutRole = []

# Code block to fetch all current Service Account details in Master Account 
logger.info("Fetching all service accounts details in Master Account")
orgClient = accountSession.client('organizations')
next_token = None
while True:
	if next_token:
		response = orgClient.list_accounts(NextToken=next_token)
	else:
		response = orgClient.list_accounts()
		
	for accounts in response['Accounts']:
		if(accounts['Id'] != '182428597156'):
			accountList.append(accounts['Id'])
		account_name = accounts['Name']
		account_id = accounts['Id']
		discovery_credentials = ''
		datacenter_type = 'AWS'
		datacenter_url = ''
		is_parent = ''
		if account_id == '182428597156': # Replace this ID with Master Account ID of McD 
			is_master = 'True'
		else:
			is_master = 'False'
		tag_resp = orgClient.list_tags_for_resource(ResourceId=account_id)
		tags = tag_resp['Tags']
		account_details = {'Name': account_name, 'Account Id': account_id, 'Discovery Credential': discovery_credentials,
						 'Datacenter Type': datacenter_type, 'Datcenter URL': datacenter_url, 'Parent Account': is_parent,
						  'Is Master Account': is_master, 'Tags' : str(tags)}
		serviceAccounts.append(account_details)
		
	if 'NextToken' in response:
		next_token = response['NextToken']
	else:
		break
logger.info("fetched all service accounts from Master Account")
logger.info("Total number of Accounts : %s", len(accountList))
logger.info(accountList)
# Code block for Service account details ends here

# Main function of the script
def handler():
	try: 
		insert_service_account_details(serviceAccounts)   # Function to insert Service Accounts details in database
		# All Threads declarations 
		vpc_subnet_thread = threading.Thread(target=filter_vpc_subnet_details)
		sg_thread = threading.Thread(target=filter_sg_details)
		hw_thread = threading.Thread(target=filter_hw_details)
		lb_thread = threading.Thread(target=filter_loadbalancer_details)
		keypair_thread = threading.Thread(target=filter_cloud_keypair_details)
		cloud_db_thread = threading.Thread(target=filter_cloud_database_details)
		aws_dc_thread = threading.Thread(target=filter_aws_dc_details)
		az_thread = threading.Thread(target=filter_az_details)
		storage_volume_thread = threading.Thread(target=filtered_storage_volume_details)
		storage_mapping_thread = threading.Thread(target=filtered_storage_mapping_details)
		vm_thread = threading.Thread(target=filtered_vm_details)
		image_thread = threading.Thread(target=filter_images_details)
		ip_thread = threading.Thread(target=filter_ip_details)
		logical_datacenter_thread = threading.Thread(target=filter_logical_datacenter_details)
		lb_ip_thread = threading.Thread(target=filter_loadbalancer_ip)
		lb_service_thread = threading.Thread(target=filter_loadbalancer_service)
		nic_thread = threading.Thread(target=filter_nic_details)
		vnic_thread = threading.Thread(target=filter_vnic_details)
		subnet_endpoint_thread = threading.Thread(target=filtered_subnet_endpoint_details)
		
		# Starting 4 Threads simultaneously to keeping CPU Utilization optimal
		sg_thread.start()
		hw_thread.start()
		lb_thread.start()
		ip_thread.start()
		sg_thread.join()
		hw_thread.join()
		lb_thread.join()
		ip_thread.join()
		
		keypair_thread.start()
		image_thread.start()		
		vpc_subnet_thread.start()
		cloud_db_thread.start()
		keypair_thread.join()
		image_thread.join()
		vpc_subnet_thread.join()
		cloud_db_thread.join()

		aws_dc_thread.start()
		storage_volume_thread.start()
		storage_mapping_thread.start()
		az_thread.start()
		aws_dc_thread.join()
		storage_mapping_thread.join()
		storage_volume_thread.join()
		az_thread.join()

		logical_datacenter_thread.start()
		lb_ip_thread.start()
		lb_service_thread.start()
		vm_thread.start()
		logical_datacenter_thread.join()
		lb_ip_thread.join()
		lb_service_thread.join()
		vm_thread.join()
		
		nic_thread.start()
		vnic_thread.start()
		subnet_endpoint_thread.start()
		nic_thread.join()
		vnic_thread.join()
		subnet_endpoint_thread.join()

		servicenow_response(f"Succesfully inserted data into table. List of accounts not having MemberRole attached - {set(accountsWithoutRole)}")   # Sending successful response after complete execution of script
		logger.info(f'List of accounts not having MemberRole attached - {set(accountsWithoutRole)}')
		logger.info("AWS Discovery script ends here......................")
	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")


# List of Get functions to fetch response from AWS using API's
def get_loadbalancer_details(session, region, val):
	try:
		elb_client = session.client('elbv2', region_name=region)
		response = elb_client.describe_load_balancers()
		for i in response['LoadBalancers']:
			i['Account'] = val
			i['Region'] = region
			arn = i['LoadBalancerArn']
			elb_client = session.client('elbv2', region_name=region)
			resp = elb_client.describe_tags(ResourceArns=[arn,])
			i['Tags'] = resp['TagDescriptions'][0]['Tags']

		logger.info("cmdb_ci_cloud_load_balancer : %s : %s : Data fetched", val, region)
		return response['LoadBalancers']

	except Exception as e:
		logger.info("Error fetching ELB details in %s: %s}", region, str(e))

def get_lb_service_details(session, region, val):
	try:
		elb_client = session.client('elbv2', region_name=region)
		response = elb_client.describe_load_balancers()
		for i in response['LoadBalancers']:
			i['Account'] = val
			arn = i['LoadBalancerArn']
			elb_client = session.client('elbv2', region_name=region)
			resp = elb_client.describe_tags(ResourceArns=[arn,])
			i['Tags'] = resp['TagDescriptions'][0]['Tags']
		
		required_lb_details = []
		for lb in response['LoadBalancers']:
			name = lb['LoadBalancerName']
			arn = lb['LoadBalancerArn']
			az = lb['AvailabilityZones'][0]['ZoneName']
			account = lb['Account']
			lbname = lb['LoadBalancerName']
			lbclass = 'Load Balancer Service'
			discovery = ''
			tags = lb['Tags']

			client = session.client('elbv2', region_name=region)
			response = response = client.describe_listeners(LoadBalancerArn=arn,)
			for i in response['Listeners']:
				port = i['Port']
				protocol = i['Protocol']
			
				details = {
					'Name' : name,
					'IpAddress' : '',
					'Port' : port,
					'protocol' : protocol,
					'Class' : lbclass,
					'LoadBalancer' : lbname,
					'Discovery' : discovery,
					'az' : az,
					'region' : region,
					'Account' :account,
					'Tags' : str(tags)
				}
				required_lb_details.append(details)

		return required_lb_details
	except Exception as e:
		print(f"Error fetching ELB details in {region}: {str(e)}")

def get_subnet_details(session, region, val):
	try:
		ec2_client = session.client('ec2', region_name=region)
		response = ec2_client.describe_subnets()
		logger.info("cmdb_ci_cloud_subnet : %s : %s : Data fetched", val, region)
		return response['Subnets']

	except Exception as e:
		logger.info("Error fetching subnet details in %s : %s", region, str(e))

def get_vpc_details(session, region, val):
	try:
		ec2_client = session.client('ec2', region_name=region)
		vpcs = ec2_client.describe_vpcs()
		for i in vpcs['Vpcs']:
			i['Region'] = region
		logger.info("cmdb_ci_network : %s : %s : Data fetched", val, region)
		return  vpcs['Vpcs']

	except Exception as e:
		logger.info("Error fetching vpc details in %s : %s", region, str(e))

def get_security_group_details(session, region, val):
	try:
		ec2_client = session.client('ec2', region_name=region)
		response = ec2_client.describe_security_groups()
		for i in response['SecurityGroups']:
			i['Region'] = region
		logger.info("cmdb_ci_compute_security_group : %s : %s : Data fetched", val, region)
		return response['SecurityGroups']
	
	except Exception as e:
		logger.info("Error fetching Security Group details in %s : %s", region, str(e))

def get_all_hardware_details(session, region, val):
	try:
		ec2_client = session.client('ec2', region_name=region)  
		response = ec2_client.describe_instances()
		instance_types = response['Reservations']
		for res in instance_types:
			if res['Instances']:
				for i in res['Instances']:
					i['Account'] = val
					i['Region'] = region

		logger.info("cmdb_ci_compute_template : %s : %s : Data fetched", val, region)
		return instance_types
		
	except Exception as e:
		logger.info("Error fetching Hardware details in %s : %s}", region, str(e))
		
def get_keypair_details(session, region, val):
	try:
		keypair_client = session.client('ec2', region_name=region)  
		response = keypair_client.describe_key_pairs()
		keypair_data = response['KeyPairs']
		for i in keypair_data:
			i['Account'] = val
			i['Region'] = region

		logger.info("cmdb_ci_cloud_key_pair : %s : %s : Data fetched", val, region)
		return keypair_data
 
	except Exception as e:
		logger.info("Error fetching Cloud key Pair details in %s : %s", region, str(e))
 
def get_cloud_db_details(session, region, val):
	try:
		db_client = session.client('rds', region_name=region)  
		response = db_client.describe_db_instances()
		instance_types = response['DBInstances']
		for i in instance_types:
			i['Account'] = val
			i['Region'] = region
		logger.info("cmdb_ci_cloud_database : %s : %s : Data fetched", val, region)
		return instance_types
		
	except Exception as e:
		logger.info("Error fetching Cloud Database details in %s : %s", region, str(e))

def get_aws_dc_details(session, region, val):
	try:
		client = session.client('ec2', region_name=region)  
		zones = client.describe_availability_zones() 

		all_details = []
		for rg in zones['AvailabilityZones']:
			region = rg['ZoneName']
			
			details = {
				'Name' : region,
				'Region' : region[:-1],
				'Account' : val
			}

			all_details.append(details)

		logger.info("cmdb_ci_aws_datacenter : %s : %s : Data fetched", val, region)
		return all_details
		
	except Exception as e:
		logger.info("Error fetching AWS datacenters details in %s : %s", region, str(e))

def get_az_details(session, region, val):
	try:
		az_client = session.client('ec2', region_name=region)  
		response = az_client.describe_availability_zones()
		instance_types = response['AvailabilityZones']

		logger.info("cmdb_ci_availability_zone : %s : %s : Data fetched", val, region)
		return instance_types
		
	except Exception as e:
		logger.info("Error fetching AvailabilityZones details in %s : %s", region, str(e))

def get_storage_vol_details(session, region, val):
	try:
		st_client = session.client('ec2', region_name=region)  
		response = st_client.describe_volumes()
		instance_types = response['Volumes']
		for i in instance_types:
			i['Account'] = val
		logger.info("cmdb_ci_storage_volume : %s : %s : Data fetched", val, region)
		return instance_types
		
	except Exception as e:
		logger.info("Error fetching Storage Volumes details in %s : %s", region, str(e))

def get_storage_mapping_details(session, region, val):
	try:
		st_client = session.client('ec2', region_name=region)  
		response = st_client.describe_volumes()

		attached_volumes = []
		unattached_volumes = []

		for volume in response['Volumes']:
			if 'Attachments' in volume and volume['Attachments']:
				attached_volumes.append(volume)
			else:
				unattached_volumes.append(volume)

		for i in attached_volumes:
			i['Account'] = val

		logger.info("cmdb_ci_storage_mapping : %s : %s : Data fetched", val, region)
		return attached_volumes
		
	except Exception as e:
		logger.info("Error fetching Storage Mapping details in %s : %s", region, str(e))

def get_public_ip_addresses(session, region, val):
	try:
		ec2_client = session.client('ec2', region_name=region)
		response = ec2_client.describe_addresses()
		ip_details = []

		# Iterating through reservations and instances to get public IPs
		for instance in response['Addresses']:
				name = next((tag['Value'] for tag in instance.get('Tags', []) if tag['Key'] == 'Name'), None)
				if name == None:
					name = instance['AllocationId']
				object_id = instance['AllocationId']
				public_ip = instance.get('PublicIp', '')
				public_dns = ''
				tags = instance.get('Tags','')
				az = region
				account =  val

				details = {
				'Name': name,
				'PublicIp': public_ip,
				'PublicDns' : public_dns,
				'ObjectId' : object_id,
				'Region' : az,
				'Account' : account,
				'Tags' : str(tags)
				}
				ip_details.append(details)
		logger.info("cmdb_ci_cloud_public_ipaddress : %s : %s : Data fetched", val, region)
		return ip_details
		
	except Exception as e:
		logger.info("Error fetching Public IP details in %s : %s ", region, str(e))

def get_images_details(session, region, val):
	try:
		ec2 = session.client('ec2', region_name=region)
		response = ec2.describe_instances()

		required_images_details = []

		for image in response['Reservations']:
			instance_id = image['Instances'][0]['InstanceId']
			object_id = image['Instances'][0]['ImageId']
			try:
				key = image['Instances'][0]['KeyName']
			except Exception as e:
				key = ''
			client = session.client('ec2', region_name=region)
			response = client.describe_images(ImageIds=[object_id]) #service cafe dev
			if 'Images' in response and response['Images']:
				for i in response['Images']:
					name = i['Name']
					image_id = i['ImageId']
					os = i['PlatformDetails']
					device_type = i['RootDeviceType']
					image_type = i['ImageType']
					source = i['ImageLocation']
					host_name = ''
					try:
						tags = i['Tags']
					except Exception as e:
						tags = ''

					details = {
					'Name' : name,
					'ObjectID' : image_id,
					'Instanceid' : instance_id,
					'OS' : os,
					'DeviceType': device_type,
					'Type' : image_type,
					'Region' : region,
					'location' : source,
					'Key': key,
					'HostName' : host_name,
					'Account' : val,
					'Tags': str(tags)
					}
					required_images_details.append(details)
			else:
				details = {
				'Name' : object_id,
				'ObjectID' : object_id,
				'Instanceid' : instance_id,
				'OS' : '',
				'DeviceType': '',
				'Type' : '',
				'Region' : region,
				'location' : '',
				'Key': '',
				'HostName' : '',
				'Account' : val,
				'Tags': ''
				}
				required_images_details.append(details)

		logger.info("cmdb_ci_os_template : %s : %s : Data fetched", val, region)
		return required_images_details
			
	except Exception as e:
		logger.info("Error fetching Images details in %s : %s", region, str(e))

def get_datacenter_details(session, region, val):
	try:
		ec2 = session.client('ec2', region_name=region)
		response = ec2.describe_regions()
		data = response['Regions']
		datacenter =[]
		for dc in data:
			name = dc['RegionName']
			region = dc['RegionName']
			discovery = ''
			account = val
			dclass = 'AWS Datacenter'

			details = {
				'Name' : name,
				'Region' : region,
				'DiscoveryStatus' : discovery,
				'account' : account,
				'Class' : dclass
			}

			datacenter.append(details)
		
		logger.info("cmdb_ci_logical_datacenter : %s : %s : Data fetched", val, region)
		return datacenter
			
	except Exception as e:
		logger.info("Error Logical Datacenter details in %s : %s", region, str(e))

def get_vm_details(session, region, val):
	try:
		ec2_client = session.client('ec2', region_name=region)  
		response = ec2_client.describe_instances()
		logger.info("cmdb_ci_vm_instance : %s : %s : Data fetched", val, region)
		return response['Reservations']
		
	except Exception as e:
		logger.info("Error fetching VM Instance details in %s : %s", region, str(e))

def get_nic_details(session, region, val):
	try:
		ec2_client = session.client('ec2', region_name=region)  
		response = ec2_client.describe_network_interfaces()
		for i in response['NetworkInterfaces']:
			i['Region'] = region
			i['Account'] = val
		logger.info("cmdb_ci_nic : %s : %s : Data fetched", val, region)

		return response['NetworkInterfaces']
		
	except Exception as e:
		logger.info("Error fetching NIC details in %s : %s", region, str(e))

def get_vnic_details(session, region, val):
	try:
		ec2_client = session.client('ec2', region_name=region)  
		response = ec2_client.describe_instances()
		logger.info("cmdb_ci_endpoint_vnic : %s : %s : Data fetched", val, region)
		return response['Reservations']
		
	except Exception as e:
		logger.info("Error fetching VNIC details in %s : %s", region, str(e))

# List of filter functions to filter required details from every classes
def filter_vpc_subnet_details():
	try:
		required_subnet_details = []
		required_vpc_details = []
		all_subnet_details = []
		all_vpc_details = []

		# Code block to fetch and segregate Subnet data
		master_session = create_master_session()
		logger.info("cmdb_ci_cloud_subnet : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					subnets = get_subnet_details(assumed_session, region, val)
					all_subnet_details.extend(subnets)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		for sb in all_subnet_details:
			sb_name = next((tag['Value'] for tag in sb.get('Tags', []) if tag['Key'] == 'Name'), None)
			if sb_name == None:
				sb_name = sb['SubnetId']
			sb_id = sb['SubnetId']
			sb_state = sb['State']
			sb_cidr = sb['CidrBlock']
			account = sb['OwnerId']
			az = sb['AvailabilityZone']
			vpc = sb['VpcId']
			tags = sb.get('Tags', '')

			details = {
				'Name': sb_name,
				'State': sb_state,
				'Id': sb_id,
				'Cidr' : sb_cidr,
				'AccountID' : account,
				'az' : az,
				'region': az[:-1],
				'vpc':vpc,
				'Tags' : str(tags)
			}

			required_subnet_details.append(details)

		# Code block for fetch and segregate VPC data
		logger.info("cmdb_ci_network : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					vpcs = get_vpc_details(assumed_session, region, val)
					all_vpc_details.extend(vpcs)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		for vp in all_vpc_details:
			vpc_name = next((tag['Value'] for tag in vp.get('Tags', []) if tag['Key'] == 'Name'), None)
			if vpc_name == None:
				vpc_name = vp['VpcId']
			vpc_id = vp['VpcId']
			vpc_state = vp['State']
			vpc_cidr = vp['CidrBlock']
			account = vp['OwnerId']
			region = vp['Region']
			tags = vp.get('Tags', '')

			details = {
				'Name': vpc_name,
				'State': vpc_state,
				'Id': vpc_id,
				'Cidr' : vpc_cidr,
				'AccountID': account,
				'region': region,
				'Tags' : str(tags)
			}
			required_vpc_details.append(details)

		insert_subnet_details(required_subnet_details)
		insert_vpc_details(required_vpc_details)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}") 

def filter_loadbalancer_details():
	try: 
		required_lb_details = []
		load_balancer_details = []

		# Code block to fetch and segregate Load Balancer data
		master_session = create_master_session()
		logger.info("cmdb_ci_cloud_load_balancer : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions :
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					load_balancer = get_loadbalancer_details(assumed_session, region, val)
					load_balancer_details.extend(load_balancer)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		for lb in load_balancer_details:
			lb_name = lb.get('LoadBalancerName','')
			lb_object_id = lb.get('LoadBalancerArn','')
			lb_state = lb['State']['Code']
			lb_hosted_zone_name = lb.get('DNSName', '')
			lb_hosted_zone_id = lb.get('CanonicalHostedZoneId', '')
			lb_dns_name = lb.get('DNSName', '')
			domain = lb.get('DNSName', '')
			sg = lb.get('SecurityGroups','')
			region = lb['Region']
			account = lb['Account']
			tags = lb['Tags']
			zones = lb['AvailabilityZones']
			zone_names = [zone['ZoneName'] for zone in zones]
			subnet_ids = [zone['SubnetId'] for zone in zones]

			if len(sg) > 1:
				for j in range(len(sg)):
					for i in range(len(zone_names)):
						lb_details = {
						'Name': lb_name,
						'ObjectID':lb_object_id,
						'State' : lb_state,
						'HostedZoneName':lb_hosted_zone_name,
						'HostedZoneID': lb_hosted_zone_id,
						'DNSName': lb_dns_name,
						'Domain' : domain,
						'sg' : sg[j],
						'subnet' : subnet_ids[i],
						'account' : account,
						'region': region,
						'az' : zone_names[i],
						'tags': str(tags)
						}
						required_lb_details.append(lb_details)			
			else:
				for i in range(len(zone_names)):
					lb_details = {
					'Name': lb_name,
					'ObjectID':lb_object_id,
					'State' : lb_state,
					'HostedZoneName':lb_hosted_zone_name,
					'HostedZoneID': lb_hosted_zone_id,
					'DNSName': lb_dns_name,
					'Domain' : domain,
					'sg' : sg,
					'subnet' : subnet_ids[i],
					'account' : account,
					'region': region,
					'az' : zone_names[i],
					'tags': str(tags)
					}
					required_lb_details.append(lb_details)
 
		insert_loadbalancer_details(required_lb_details)
	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filter_sg_details():
	try:
		required_sg_details = []
		security_group_details = []

		# Code block to fetch and segregate security group details
		master_session = create_master_session()
		logger.info("cmdb_ci_compute_security_group : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					security_group = get_security_group_details(assumed_session, region, val)
					security_group_details.extend(security_group)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		for sg in security_group_details:
			sg_name = next((tag['Value'] for tag in sg.get('Tags', []) if tag['Key'] == 'Name'), None)
			if sg_name == None:
				sg_name = sg['GroupName']
			sg_id = sg['GroupId']
			sg_state = ''
			account = sg['OwnerId']
			network = sg['VpcId']
			region = sg['Region']
			tags = sg.get('Tags', '')

			details = {
				'Name': sg_name,
				'Id': sg_id,
				'State' : sg_state,
				'Region': region,
				'Account' : account,
				'Network' : network,
				'Tags' : str(tags)
			}
			required_sg_details.append(details)

		insert_security_group_details(required_sg_details)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filter_images_details():
	try:
		images_details = []

		# Code block to fetch and segregate Images details
		master_session = create_master_session()
		logger.info("cmdb_ci_os_template : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					images = get_images_details(assumed_session, region, val)
					images_details.extend(images)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		insert_image_details(images_details)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filter_hw_details():
	try:
		required_hardware_details = []
		all_hardware_details = []

		# Code block to fetch Hardware details
		master_session = create_master_session()
		logger.info("cmdb_ci_compute_template : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					hardware = get_all_hardware_details(assumed_session, region, val)
					all_hardware_details.extend(hardware)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)
	
		for hw in all_hardware_details:
			hw_name = hw['Instances'][0]['InstanceType']
			instance_id = hw['Instances'][0]['InstanceId']
			ec2_client = boto3.client('ec2', region_name='us-east-1')
			response = ec2_client.describe_instance_types(InstanceTypes=[hw_name])
			for i in response['InstanceTypes']:
				if (i['InstanceStorageSupported'] == True) :
					hw_storage = i['InstanceStorageInfo']['TotalSizeInGB']
				else:
					hw_storage = ''
				hw_memory = i['MemoryInfo']['SizeInMiB']
				hw_vcpu = i['VCpuInfo']['DefaultVCpus']
			account = hw['Instances'][0]['Account']
			region = hw['Instances'][0]['Placement']['AvailabilityZone'][:-1]
			try:
				tags = hw['Instances'][0]['Tags']
			except Exception as e:
				tags = ''
			

			details = {
				'Name' : hw_name,
				'instance_id':instance_id,
				'Vcpu' : hw_vcpu,
				'Memory' : hw_memory,
				'Storage' : hw_storage,
				'Region' : region,
				'Account' : account,
				'Tags' : str(tags)
			}

			required_hardware_details.append(details)
		
		insert_hardware_details(required_hardware_details)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filter_cloud_keypair_details():
	try:
		required_key = []
		key_pair_dtls = []

		# Code block to fetch Cloud Keypair details
		master_session = create_master_session()
		logger.info("cmdb_ci_cloud_key_pair : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					key_pair = get_keypair_details(assumed_session, region, val)
					key_pair_dtls.extend(key_pair)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		for kp in key_pair_dtls:
			name = kp.get('KeyName')
			object_id = kp.get('KeyPairId')
			finger_print = kp.get('KeyFingerprint')
			account_id = kp.get('Account')
			region =kp.get('Region')
			tags = kp.get('Tags', '')
 
			details = {
				'Name': name,
				'Object_ID':object_id,
				'Finger_Print' : finger_print,
				'region' : region,
				'Account' : account_id,
				'Tags' : str(tags)
			}
 
			required_key.append(details)
		
		insert_key_pair_details(required_key)
	except Exception as e:
		logger.info(e) 
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filter_cloud_database_details():
	try:
		cloud_db_details = []
		cloud_database_details = []

		# Code bolck to fetch Cloud Databse details
		master_session = create_master_session()
		logger.info("cmdb_ci_cloud_database : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)
 
				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					cloud_db = get_cloud_db_details(assumed_session, region, val)
					cloud_database_details.extend(cloud_db)
 
			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		for res in cloud_database_details:
			try:
				name = res['DBInstanceIdentifier']
			except Exception as e:
				name = ''
			try:
				cat = res['Engine']
			except Exception as e:
				cat = ''
			try:
				ver = res['EngineVersion']
			except Exception as e:
				ver = ''
			try:
				db_class = res['DBInstanceClass']
			except Exception as e:
				db_class = ''
			port = res['Endpoint']['Address']
			network = res['DBSubnetGroup']['VpcId']
			az = res['AvailabilityZone']
			account = res['Account']
			tags = res['TagList']
 
			cld_dtbs_dtls = {
				'Name': name,
				'Category':cat,
				'Version' : ver,
				'Class': db_class,
				'port' : port,
				'network' : network,
				'az' : az,
				'region' : az[:-1],
				'Account' : account,
				'Tags' : str(tags)
			}
 
			cloud_db_details.append(cld_dtbs_dtls)
		insert_cloud_db_details(cloud_db_details)
 
	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filter_aws_dc_details():
	try:
		aws_rzn = []
		
		# Code block to fetch AWS Datacenter details
		master_session = create_master_session()
		logger.info("cmdb_ci_aws_datacenter : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					data = get_aws_dc_details(assumed_session, region, val)
					aws_rzn.extend(data)


			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		insert_aws_dc_details(aws_rzn)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filter_az_details():
	try:
		avail_zones_details = []

		# Code block to fetch Availiability Zones details
		master_session = create_master_session()
		logger.info("cmdb_ci_availability_zone : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					avl_zone = get_az_details(assumed_session, region, val)
					for az in avl_zone:
						name = az['ZoneName']

						details = {
							'Name': name,
							'Datacenter' : name,
							'Account' : val,
							'Region' : name[:-1],
							'status' : '',
							'Class' : 'AWS Datacenter'
							}

						avail_zones_details.append(details)
					
			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		insert_az_details(avail_zones_details)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filter_ip_details():
	try:
		ip_details = []

		# Code block to fetch IP Address details
		master_session = create_master_session()
		logger.info("cmdb_ci_cloud_public_ipaddress : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					ip = get_public_ip_addresses(assumed_session, region, val)
					ip_details.extend(ip)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		insert_ip_deatils(ip_details)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filtered_storage_volume_details():
	try:
		strg_vol_details = []
		strg_vol_dtls = []

		# Code block to fetch Storage Volumes details
		master_session = create_master_session()
		logger.info("cmdb_ci_storage_volume : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					strg_vol = get_storage_vol_details(assumed_session, region, val)
					strg_vol_dtls.extend(strg_vol)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		for sv in strg_vol_dtls:
			name = next((tag['Value'] for tag in sv.get('Tags', []) if tag['Key'] == 'Name'), None)
			if name == None:
				name =sv['VolumeId']
			obj_id = sv['VolumeId']
			state = sv['State']
			size = sv['Size']
			strg_type = sv['VolumeType']
			account = sv['Account']
			az = sv['AvailabilityZone']
			tags = sv.get('Tags', '')
			try:
				instance_id = sv['Attachments'][0]['InstanceId']
			except Exception as e:
				instance_id = ""

			details = {
				'Name': name,
				'Object_ID':obj_id,
				'instance': instance_id,
				'State' : state,
				'Size':size,
				'Storage_Type':strg_type,
				'Account' : account,
				'Region' : az[:-1],
				'az': az,
				'Tags' : str(tags)
			}

			strg_vol_details.append(details)
		
		insert_storage_volume_details(strg_vol_details)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filtered_storage_mapping_details():
	try:
		storage_mapping_data = []
		strg_map_dtls = []

		# Code block to fetch Storage Mapping details
		master_session = create_master_session()
		logger.info("cmdb_ci_storage_mapping : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					strg_map = get_storage_mapping_details(assumed_session, region, val)
					strg_map_dtls.extend(strg_map)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)
	
		for mp in strg_map_dtls:
			name = next((tag['Value'] for tag in mp.get('Tags', []) if tag['Key'] == 'Name'), None)
			if name == None:
				name = mp['VolumeId']
			obj_id = mp['VolumeId']
			map_type = ''
			host = ''
			mount_point = mp['Attachments'][0]['Device']
			account = mp['Account']
			az = mp['AvailabilityZone']
			tags = mp.get('Tags','')
			try:
				vm_id = mp['Attachments'][0]['InstanceId']
			except Exception as e:
				vm_id = ""

			details = {
				'Name': name,
				'Object_ID':obj_id,
				'instance' : vm_id,
				'Mapping_Type' : map_type,
				'Host':host,
				'Mount_Point':mount_point,
				'Account' : account,
				'Region': az[:-1],
				'az':az,
				'Tags' : str(tags)
			}

			storage_mapping_data.append(details)
		
		insert_storage_mapping_details(storage_mapping_data)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")
	
def filter_logical_datacenter_details():
	try:
		dc_details = []

		# Code block to fetch Logical Datacenters details
		master_session = create_master_session()
		logger.info("cmdb_ci_logical_datacenter : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					dc = get_datacenter_details(assumed_session, region, val)
					dc_details.extend(dc)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		insert_logical_datacenter_deatils(dc_details)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filter_loadbalancer_ip():
	try:
		all_lb_details = []
		required_lb_details = []

		# Code block to fetch Loadbalancer IP details
		master_session = create_master_session()
		logger.info("cmdb_ci_cloud_lb_ipaddress : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					lb = get_loadbalancer_details(assumed_session, region, val)
					all_lb_details.extend(lb)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		for lb in all_lb_details:
			name = lb['LoadBalancerName']
			ip_type = lb['IpAddressType']
			az = lb['AvailabilityZones'][0]['ZoneName']
			account = lb['Account']
			tags = lb['Tags']

			details = {
				'Name' : name,
				'IpAddressType' : ip_type,
				'az' : az,
				'region' : az[:-1],
				'Account' : account,
				'Tags': str(tags)
			}

			required_lb_details.append(details)

		insert_lb_ip_details(required_lb_details)
	except Exception as e:
			logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
			servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filter_loadbalancer_service():
	try:
		all_lb_details = []

		# Code block to fetch Loadbalancer Service details
		master_session = create_master_session()
		logger.info("cmdb_ci_lb_service : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					lb = get_lb_service_details(assumed_session, region, val)
					all_lb_details.extend(lb)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		insert_lb_service_details(all_lb_details)
		
	except Exception as e:
			logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
			servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filtered_vm_details():
	try:
		vm_instance_details = []
		vm_instance = []

		# Code block to fetch VM Instance details
		master_session = create_master_session()
		logger.info("cmdb_ci_vm_instance : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					vm_ins = get_vm_details(assumed_session, region, val)
					vm_instance.append(vm_ins)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		for reservation in vm_instance:
			for res in reservation:
				if res['Instances']:
					for instance in res['Instances']:
						name = next((tag['Value'] for tag in instance.get('Tags', []) if tag['Key'] == 'Name'), None)
						if name == None:
							name = instance.get('InstanceId', '')
						try:
							az = instance['Placement']['AvailabilityZone']
						except Exception as e:
							az = ""
						vm_id = instance.get('InstanceId', '')
						network_interfaces = instance.get('NetworkInterfaces', [])
						for network_interface in network_interfaces:
							owner_id = network_interface.get('OwnerId', None)
						obj = instance['InstanceId']
						public_ip = instance.get('PrivateIpAddress', '')
						state = instance['State']['Name']
						cpu = instance['CpuOptions']['CoreCount']
						tags = instance.get('Tags', '')
						image = instance.get('ImageId', '')
						hw = instance.get('InstanceType', '')
						nw_nics = len(instance['NetworkInterfaces'])
						disks = len(instance['BlockDeviceMappings'])
						subnet = instance.get('SubnetId', '')
						vpc = instance.get('VpcId','')

						details = {
							'Name': name,
							'Aws_Account_id': owner_id,
							'Provider': "AWS",
							'region': az[:-1],
							'az': az,
							'Instanceid': vm_id,
							'ObjectID' : obj,
							'imageId' : image,
							'hw': hw,
							'Memory':hw,
							'subnet' : subnet,
							'vpc' : vpc,
							'disks':disks,
							'Network_Adapter':nw_nics,
							'Os': '',
							'Ip_Address': public_ip,
							'State' : state,
							'cpu' : cpu,
							'Tags' : str(tags)
							}

						vm_instance_details.append(details)

		insert_vm_details(vm_instance_details)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filter_nic_details():
	try:
		network_details = []
		nic_details = []

		# Code block to fetch Network Interfaces details
		master_session = create_master_session()
		logger.info("cmdb_ci_nic : Master session created")
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					net = get_nic_details(assumed_session, region, val)
					if net != None:
						network_details.extend(net)
					
			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		for i in network_details:
			name = next((tag['Value'] for tag in i.get('TagSet', []) if tag['Key'] == 'Name'), None)
			if name == None:
				name = i.get('NetworkInterfaceId', '')
			net_id = i.get('NetworkInterfaceId','')
			account = i.get('Account','')
			az = i.get('AvailabilityZone','')
			at = i.get('Attachment', '')
			try:
				vm = at['InstanceId']
			except Exception as e:
				vm = ''
			ass = i.get('Association', '')
			try:
				public_ip = ass['PublicIp']
			except Exception as e:
				public_ip = ''
			private_ip = i.get('PrivateIpAddress','')
			state = i.get('Status','')
			tags = i.get('TagSet','')
			subnet = i.get('SubnetId','')

			details = {
						'Name': name,
						'Account': account,
						'region': az[:-1],
						'az': az,
						'Instanceid': vm,
						'publicip': public_ip,
						'privateip' : private_ip,
						'State' : state,
						'vnic': net_id,
						'subnet': subnet,
						'Tags' : str(tags)
						}
			
			nic_details.append(details)
		
		insert_nic_details(nic_details)
	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filter_vnic_details():
	try:
		vnic_details = []
		vnic_instance = []
		
		# Code block to fetch Virtual Network Interfaces details
		master_session = create_master_session()
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					vnic = get_vnic_details(assumed_session, region, val)
					vnic_instance.append(vnic)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		for reservation in vnic_instance:
			for res in reservation:
				if res['Instances']:
					for instance in res['Instances']:
						inst_id = res['Instances'][0]['InstanceId']
						network_interfaces = instance.get('NetworkInterfaces', [])
						for network_interface in network_interfaces:
							nw_id = network_interface.get('NetworkInterfaceId', '')
						public_ip = instance.get('PrivateIpAddress', '')

						details = {
							'Host': inst_id,
							'Name': nw_id,
							'Ip_Address': public_ip,
							'Object_ID' : nw_id
							}

						vnic_details.append(details)

		insert_vnic_details(vnic_details)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def filtered_subnet_endpoint_details():
	try:
		required_subnet_endpoint_details = []
		all_endpoint_subnet_details = []

		# Code block to fetch Subnet Endpoints details
		master_session = create_master_session()
		for val in accountList:
			try:
				role = f'arn:aws:iam::{val}:role/MemberRole'
				assumed_session, credentials = assume_new_session(master_session, role)

				for region in all_regions:
					if (credentials['Expiration'].replace(tzinfo=timezone.utc) - datetime.utcnow().replace(tzinfo=timezone.utc)).total_seconds() < 300: 
						assumed_session, credentials = assume_new_session(master_session, role)
					subnets = get_subnet_details(assumed_session, region, val)
					all_endpoint_subnet_details.extend(subnets)

			except Exception as e:
				logger.info(f"{const.ROLE_ERROR} {val} : {str(e)}")
				accountsWithoutRole.append(val)

		for sb in all_endpoint_subnet_details:
			name = sb.get('SubnetId', '')

			details = {
				'Name': name,
				'Manufacturer': "",
				'Location': "",
				'Description' : '',
				'Class' : "Subnet Endpoint",
				'Updated' : '',
				'Mainteinance_Schedule' : ""
			}

			required_subnet_endpoint_details.append(details)
		insert_subnet_endpoint_details(required_subnet_endpoint_details)

	except Exception as e:
		logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

# List of functions to insert required data into database
def insert_vpc_details(vpc_details):
	try:
		logger.info("cmdb_ci_network : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_network'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)

		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_network (
				Name varchar(50),
				State varchar(50),
				Object_ID varchar(50),
				CIDR varchar(50),
				Account_ID varchar(50),
				Region varchar(50),
				Tags varchar(5000)
			);""" 
		cursor.execute(create_table)

		for vpc in vpc_details:
			print(vpc)
			insert_query = """
				INSERT INTO cmdb_ci_network (Name, State, Object_ID, CIDR, Account_ID, Region, Tags)
				VALUES (%s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (vpc['Name'], vpc['State'], vpc['Id'], vpc['Cidr'],vpc['AccountID'], vpc['region'], vpc['Tags']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")

		logger.info("cmdb_ci_network : Successfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_network : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_subnet_details(subnet_details):
	try:
		logger.info("cmdb_ci_cloud_subnet : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_cloud_subnet'
		cursor = connection.cursor() 

		check_table_in_database(table_name, cursor)

		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_cloud_subnet (
				Name varchar(150),
				State varchar(50),
				Object_ID varchar(50),
				CIDR varchar(50),
				Network_Object_ID varchar(50),
				Account_ID varchar(50),
				Availability_Zone varchar(50),
				Region varchar(50),
				Tags varchar(5000)
				);"""
		cursor.execute(create_table)

		for sb in subnet_details:
			print(sb)
			insert_query = """
				INSERT INTO cmdb_ci_cloud_subnet (Name, State, Object_ID, CIDR, Network_Object_ID, Account_ID, Availability_Zone, Region, Tags)
				VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (sb['Name'], sb['State'], sb['Id'], sb['Cidr'], sb['vpc'], sb['AccountID'], sb['az'], sb['region'], sb['Tags']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_cloud_subnet : Succesfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_cloud_subnet : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_loadbalancer_details(elb_details):
	try:
		logger.info("cmdb_ci_cloud_load_balancer : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_cloud_load_balancer'
		cursor = connection.cursor() 

		check_table_in_database(table_name, cursor)

		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_cloud_load_balancer (
				Name varchar(150),
				Object_ID varchar(150),
				State varchar(50),
				Canonical_Hosted_Zone_Name varchar(250),
				Canonical_Hosted_Zone_ID varchar(50),
				DNS_Name varchar(250),
				Fully_Qualified_Domain_Name varchar(250),
				Security_Group_ID varchar(250),
				Subnet_ID varchar(50),
				Region varchar(50),
				Availability_Zone varchar(50),
				Account_ID varchar(50),
				Tags varchar(5000)
			);"""
		cursor.execute(create_table)

		for elb in elb_details:
			print(elb)
			try:
				insert_query = """
					INSERT INTO cmdb_ci_cloud_load_balancer (Name, Object_ID, State, Canonical_Hosted_Zone_Name, Canonical_Hosted_Zone_ID, DNS_Name, Fully_Qualified_Domain_Name,
					  Security_Group_ID, Subnet_ID, Region, Availability_Zone, Account_ID, Tags)
					VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s);
				"""
			except pymysql.Error as e:
				print(e)
			try:
				cursor.execute(insert_query, (elb['Name'], elb['ObjectID'], elb['State'], elb['HostedZoneName'], elb['HostedZoneID'], elb['DNSName'], elb['Domain'], elb['sg'], elb['subnet'],
								   elb['region'], elb['az'], elb['account'], elb['tags']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")

		logger.info("cmdb_ci_cloud_load_balancer : Succesfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_cloud_load_balancer : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_security_group_details(sg_details):
	try:
		logger.info("cmdb_ci_compute_security_group : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_compute_security_group'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)

		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_compute_security_group (
				Name varchar(50),
				Object_ID varchar(50),
				State varchar(50),
				Region varchar(50),
				Account_ID varchar(50),
				Cloud_Networks_Object_ID varchar(50),
				Tags varchar(5000)
			);""" 
		cursor.execute(create_table)

		for sg in sg_details:
			print(sg)
			insert_query = """
				INSERT INTO cmdb_ci_compute_security_group (Name, Object_ID, State, Region,  Account_ID, Cloud_Networks_Object_ID, Tags)
				VALUES (%s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (sg['Name'], sg['Id'], sg['State'], sg['Region'], sg['Account'], sg['Network'], sg['Tags']))
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_compute_security_group : Succesfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_compute_security_group : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_hardware_details(hw_details):
	try:
		logger.info("cmdb_ci_compute_template : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_compute_template'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)
		
		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_compute_template (
				Name varchar(50),
				Instance_ID varchar(50),
				vCPUs varchar(50),
				Memory_MB varchar(50),
				Local_Storage_GB varchar(50),
				Region varchar(50),
				Account_ID varchar(50),
				Tags varchar(5000)
			);"""
		cursor.execute(create_table)

		for hw in hw_details:
			print(hw)
			insert_query = """
				INSERT INTO cmdb_ci_compute_template (Name, Instance_ID, vCPUs, Memory_MB, Local_Storage_GB, Region, Account_ID, Tags)
				VALUES (%s, %s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (hw['Name'], hw['instance_id'], hw['Vcpu'], hw['Memory'], hw['Storage'], hw['Region'], hw['Account'], hw['Tags']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_compute_template : Successfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_compute_template : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_service_account_details(service_details):
	try:
		logger.info("cmdb_ci_cloud_service_account : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_cloud_service_account'

		cursor = connection.cursor() 

		check_table_in_database(table_name, cursor)

		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_cloud_service_account (
				Name varchar(50),
				Account_Id varchar(50),
				Discovery_credentials varchar(150),
				Datacenter_Type varchar(150),
				Datacenter_URL varchar(150),
				Parent_Account varchar(50),
				Is_master_account varchar(50),
				Tags varchar(5000)
			);
			"""
		cursor.execute(create_table)

		for sv in service_details:
			print(sv)
			insert_query = """
				INSERT INTO cmdb_ci_cloud_service_account (Name, Account_Id, Discovery_credentials, Datacenter_Type, Datacenter_URL, Parent_Account, Is_master_account, Tags )
				VALUES (%s, %s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (sv['Name'], sv['Account Id'], sv['Discovery Credential'], sv['Datacenter Type'], sv['Datcenter URL'], 
											  sv['Parent Account'], sv['Is Master Account'], sv['Tags']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_cloud_service_account : Successfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_cloud_service_account : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_key_pair_details(keypair):
	try:
		logger.info("cmdb_ci_cloud_key_pair : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_cloud_key_pair'
		cursor = connection.cursor() 

		check_table_in_database(table_name, cursor)

		create_table ="""
			CREATE TABLE IF NOT EXISTS cmdb_ci_cloud_key_pair (
				Name varchar(50),
				Object_ID varchar(50),
				Finger_Print varchar(150),
				Region varchar(50),
				Account_ID varchar(50),
				Tags varchar(5000)
			);"""
		cursor.execute(create_table)

		for kp in keypair:
			print(kp)
			try:
				insert_query = """
					INSERT INTO cmdb_ci_cloud_key_pair (Name, Object_ID, Finger_Print, Region, Account_ID, Tags)
					VALUES (%s, %s, %s, %s, %s, %s);
				"""
				cursor.execute(insert_query, (kp['Name'], kp['Object_ID'], kp['Finger_Print'], kp['region'], kp['Account'], kp['Tags']))
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_cloud_key_pair : Succesfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_cloud_key_pair : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_cloud_db_details(cld_db_details):
	try:
		logger.info("cmdb_ci_cloud_database : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_cloud_database'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)

		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_cloud_database (
				Name varchar(50),
				Category varchar(50),
				Version varchar(50),
				Class varchar(50),
				Port varchar(150),
				Network_ID varchar(50),
				Availability_Zone varchar(50),
				Region varchar(50),
				Account_ID varchar(50),
				Tags varchar(5000)
			);"""
		cursor.execute(create_table)

		for cldb in cld_db_details:
			print(cldb)
			insert_query = """
				INSERT INTO cmdb_ci_cloud_database (Name, Category, Version, Class, Port, Network_ID, Availability_Zone, Region, Account_ID, Tags )
				VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (cldb['Name'], cldb['Category'], cldb['Version'], cldb['Class'], cldb['port'], cldb['network'], cldb['az'], cldb['region'], cldb['Account'], cldb['Tags']))

			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_cloud_database : Succesdfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_cloud_database : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_aws_dc_details(aws_dc_details):
	try:
		logger.info("cmdb_ci_aws_datacenter : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_aws_datacenter'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)
		
		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_aws_datacenter (
				Name varchar(50),
				Region varchar(50),
				Account_ID varchar(50),
				Discovery_Status varchar(50),
				Class varchar(50)
			);""" 
		cursor.execute(create_table)

		for dc in aws_dc_details:
			print(dc)
			discovery_status = ''
			d_class = "AWS Datacenter"

			insert_query = """
				INSERT INTO cmdb_ci_aws_datacenter (Name, Region, Account_ID, Discovery_Status, Class)
				VALUES (%s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (dc['Name'], dc['Region'], dc['Account'], discovery_status, d_class))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_aws_datacenter : Successfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_aws_datacenter : Error in inserting data into database : %s ", str(e))

def insert_az_details(az_details):
	try:
		logger.info("cmdb_ci_availability_zone : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_availability_zone'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)
		
		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_availability_zone (
				Name varchar(50),
				Region varchar(50),
				Account_ID varchar(50),
				Datacenter varchar(50),
				Discovery_Status varchar(50),
				Class varchar(50)
			);"""
		cursor.execute(create_table)

		for az in az_details:
			print(az)
			insert_query = """
				INSERT INTO cmdb_ci_availability_zone (Name, Region, Account_ID, Datacenter, Discovery_Status, Class)
				VALUES (%s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (az['Name'], az['Region'], az['Account'], az['Datacenter'], az['status'], az['Class']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_availability_zone : Inserting data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_availability_zone : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_storage_volume_details(strg_vol_details):
	try:
		logger.info("cmdb_ci_storage_volume : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_storage_volume'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)

		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_storage_volume (
				Name varchar(50),
				Object_ID varchar(150),
				Instance_ID varchar(50),
				State varchar(50),
				Size varchar(50),
				Storage_type varchar(50),
				Account_ID varchar(50),
				Region varchar(50),
				Availability_Zone varchar(50),
				Tags varchar(5000) 
			);"""
		cursor.execute(create_table)

		for vol in strg_vol_details:
			print(vol)
			insert_query = """
				INSERT INTO cmdb_ci_storage_volume (Name, Object_ID, Instance_ID, State, Size, Storage_Type, Account_ID, Region, Availability_Zone, Tags)
				VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (vol['Name'], vol['Object_ID'], vol['instance'], vol['State'], vol['Size'], vol['Storage_Type'], vol['Account'],vol['Region'], vol['az'], vol['Tags']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_storage_volume : Successfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_storage_volume : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_storage_mapping_details(strg_map_details):
	try:
		logger.info("cmdb_ci_storage_mapping : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_storage_mapping'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)
		
		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_storage_mapping (
				Name varchar(50),
				Object_ID varchar(150),
				Instance_ID varchar(50),
				Mapping_Type varchar(50),
				Host varchar(50),
				Mount_Point varchar(50),
				Account_ID varchar(150),
				Region varchar(50),
				Availability_Zone varchar(50),
				Tags varchar(5000)
			);"""
		cursor.execute(create_table)

		for mp in strg_map_details:
			print(mp)
			insert_query = """
				INSERT INTO cmdb_ci_storage_mapping (Name, Object_ID, Instance_ID, Mapping_Type, Host, Mount_Point, Account_ID, Region, Availability_Zone, Tags)
				VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (mp['Name'], mp['Object_ID'], mp['instance'], mp['Mapping_Type'], mp['Host'], mp['Mount_Point'], mp['Account'], mp['Region'], mp['az'], mp['Tags']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_storage_mapping : Inserting data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_storage_mapping : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_image_details(os_details):
	try:
		logger.info("cmdb_ci_os_template : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_os_template'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)

		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_os_template (
				Name varchar(150),
				Object_ID varchar(100),
				Instance_ID varchar(100),
				Guest_OS varchar(50),
				Root_Device_Type varchar(50),
				Image_Type varchar(50),
				Image_Source varchar(150),
				Region varchar(50),
				Infuse_Key varchar(50),
				Update_Host_Name varchar(50),
				Account_ID varchar(50),
				Tags varchar(5000)
			);"""
		cursor.execute(create_table)
		
		for os in os_details:
			print(os)
			insert_query = """
				INSERT INTO cmdb_ci_os_template (Name, Object_ID, Instance_ID, Guest_OS, Root_Device_Type, Image_Type, Image_Source, Region, Infuse_Key, Update_Host_Name, Account_ID, Tags )
				VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (os['Name'], os['ObjectID'], os['Instanceid'], os['OS'], os['DeviceType'], os['Type'], os['location'],
								os['Region'], os['Key'], os['HostName'], os['Account'], os['Tags']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_os_template : Successfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_os_template : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")
	
def insert_ip_deatils(ip_details):
	try:
		logger.info("cmdb_ci_cloud_public_ipaddress : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_cloud_public_ipaddress'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)
		
		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_cloud_public_ipaddress (
				Name varchar(100),
				Public_IP_Address varchar(50),
				Public_DNS varchar(150),
				Object_ID varchar(150),
				Account_ID varchar(50),
				Region varchar(50),
				Tags varchar(5000)
			);"""
		cursor.execute(create_table)
		
		for ip in ip_details:
			print(ip)
			insert_query = """
				INSERT INTO cmdb_ci_cloud_public_ipaddress (Name, Public_IP_Address, Public_DNS, Object_ID, Account_ID, Region, Tags)
				VALUES (%s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (ip['Name'], ip['PublicIp'], ip['PublicDns'], ip['ObjectId'], ip['Account'], 
								  ip['Region'], ip['Tags']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_cloud_public_ipaddress : Successfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_cloud_public_ipaddress : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_logical_datacenter_deatils(dc_details):
	try:
		logger.info("cmdb_ci_logical_datacenter : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_logical_datacenter'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)
		
		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_logical_datacenter (
				Name varchar(50),
				Region varchar(50),
				Discovery_Status varchar(50),
				Account_ID varchar(50),
				Class varchar(50)
			);"""
		cursor.execute(create_table)
		
		for dc in dc_details:
			print(dc)
			insert_query = """
				INSERT INTO cmdb_ci_logical_datacenter (Name, Region, Discovery_Status, Account_ID, Class)
				VALUES (%s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (dc['Name'], dc['Region'], dc['DiscoveryStatus'], dc['account'], dc['Class']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_logical_datacenter : Successfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_logical_datacenter : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_lb_ip_details(lb_details):
	try:
		logger.info("cmdb_ci_cloud_lb_ipaddress : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_cloud_lb_ipaddress'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)

		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_cloud_lb_ipaddress (
				Name varchar(50),
				IPAddress_Type varchar(50),
				Availability_Zone varchar(50),
				Region varchar(50),
				Account_ID varchar(50),
				Tags varchar(5000)
			);"""

		cursor.execute(create_table)
		
		for lb in lb_details:
			print(lb)
			insert_query = """
				INSERT INTO cmdb_ci_cloud_lb_ipaddress (Name, IPAddress_Type, Availability_Zone, Region, Account_ID, Tags)
				VALUES (%s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (lb['Name'], lb['IpAddressType'], lb['az'], lb['region'], lb['Account'], lb['Tags']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_cloud_lb_ipaddress : Succesfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_cloud_lb_ipaddress : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")
	
def insert_lb_service_details(lb_details):
	try:
		logger.info("cmdb_ci_lb_service : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_lb_service'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)
		
		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_lb_service (
				Name varchar(250),
				IP_Address varchar(50),
				Port varchar(50),
				Protocol varchar(50),
				Class varchar(50),
				Load_balancer varchar(250),
				Most_recent_discovery varchar(50),
				Availability_Zone varchar(50),
				Region varchar(50),
				Account_ID varchar(50),
				Tags varchar(5000)
			);"""

		cursor.execute(create_table)
		
		for lb in lb_details:
			print(lb)
			insert_query = """
				INSERT INTO cmdb_ci_lb_service (Name, IP_Address, Port, Protocol, Class, Load_balancer, Most_recent_discovery, Availability_Zone, Region, Account_ID, Tags)
				VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (lb['Name'], lb['IpAddress'], lb['Port'], lb['protocol'], lb['Class'], lb['LoadBalancer'], lb['Discovery'], lb['az'], lb['region'], lb['Account'], lb['Tags']))
			
			except pymysql.Error as e:
				print(f"Error: {e}")

		logger.info("cmdb_ci_lb_service : Inserting data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_lb_service : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_vm_details(vm_ins_details):
	try:
		logger.info("cmdb_ci_vm_instance : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_vm_instance'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)
		
		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_vm_instance (
				Name varchar(50),
				AWS_Account_ID varchar(50),
				Provider varchar(50),
				Region varchar(50),
				Availability_Zone varchar(50),
				Instance_ID varchar(50),
				Object_ID varchar(50),
				Image_ID varchar(50),
				Hardware_Type varchar(50),
				Memory varchar(50),
				Subnet_ID varchar(50),
				Vpc_ID varchar(50),
				Disks varchar(50),
				Network_Adapter varchar(50),
				Os varchar(50),
				State varchar(50),
				CPU varchar(50),
				Ip_Address varchar(50),
				Tags varchar(5000)
			);"""
		cursor.execute(create_table)

		for vm in vm_ins_details:
			print(vm)
			insert_query = """
				INSERT INTO cmdb_ci_vm_instance (Name, Aws_Account_id, Provider, Region, Availability_Zone, 
				Instance_ID, Object_ID, Image_ID, Hardware_Type, Memory, Subnet_ID, Vpc_ID, Disks, Network_Adapter, Os, State, CPU,  Ip_Address, Tags)
				VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s,  %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (vm['Name'], vm['Aws_Account_id'], vm['Provider'], vm['region'], vm['az'], vm['Instanceid'], vm['ObjectID'],
								   vm['imageId'], vm['hw'], vm['Memory'], vm['subnet'], vm['vpc'], vm['disks'], vm['Network_Adapter'], vm['Os'], vm['State'], vm['cpu'], vm['Ip_Address'], vm['Tags']))
			
			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")
		
		logger.info("cmdb_ci_vm_instance : Successfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_vm_instance : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_nic_details(nic_details):
	try:
		logger.info("cmdb_ci_nic : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_nic'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)
		
		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_nic (
				Name varchar(50),
				Public_IP varchar(50),
				Private_IP varchar(50),
				State varchar(50),
				Account_ID varchar(50),
				Region varchar(50),
				Instance_ID varchar(50),
				Vnic_ID varchar(50),
				Subnet_ID varchar(50),
				Tags varchar(5000)
			);"""	
		cursor.execute(create_table)

		for n in nic_details:
			print(n)
			insert_query = """
				INSERT INTO cmdb_ci_nic (Name, Public_IP, Private_IP, State, Account_ID, Region, Instance_ID, Vnic_ID, Subnet_ID, Tags)
				VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (n['Name'], n['publicip'], n['privateip'], n['State'], n['Account'], n['region'], n['Instanceid'],
									n['vnic'], n['subnet'], n['Tags']))

			except pymysql.Error as e:
				logger.info(f"{const.ERROR_MESSAGE} :  {str(e)}")

		logger.info("cmdb_ci_nic : Successfully inserted data into table")
		connection.commit()
		connection.close()

	except Exception as e:
		logger.info("cmdb_ci_nic : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_vnic_details(vnic_details):
	try:
		logger.info("cmdb_ci_endpoint_vnic : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_endpoint_vnic'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)

		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_endpoint_vnic (
				Host varchar(50),
				Name varchar(50),
				Ip_Address varchar(50),
				Object_ID varchar(50)
			);"""
		cursor.execute(create_table)

		for vm in vnic_details:
			print(vm)
			insert_query = """
				INSERT INTO cmdb_ci_endpoint_vnic (Host, Name, Ip_Address, Object_ID)
				VALUES (%s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (vm['Host'], vm['Name'], vm['Ip_Address'], vm['Object_ID']))
			
			except pymysql.Error as e:
				print(f"Error: {e}")
		
		logger.info("cmdb_ci_endpoint_vnic : Successfully inserted data into table")
		connection.commit()
		connection.close()
	except Exception as e:
		logger.info("cmdb_ci_endpoint_vnic : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

def insert_subnet_endpoint_details(subnet_endpoint_details):
	try:
		logger.info("cmdb_ci_endpoint_subnet : Inserting data into table")
		connection = pymysql.connect(host=db_host, user=db_user, password=db_pass, db=db_name, cursorclass=pymysql.cursors.DictCursor)
		table_name = 'cmdb_ci_endpoint_subnet'
		cursor =  connection.cursor() 

		check_table_in_database(table_name, cursor)

		create_table = """
			CREATE TABLE IF NOT EXISTS cmdb_ci_endpoint_subnet (
				Name varchar(100),
				Manufacturer varchar(50),
				Location varchar(50),
				Description varchar(50),
				Class varchar(50),
				Updated varchar(50),
				Mainteinance_Schedule varchar(50)
			);"""
		cursor.execute(create_table)

		for sub in subnet_endpoint_details:
			print(sub)
			insert_query = """
				INSERT INTO cmdb_ci_endpoint_subnet (Name, Manufacturer, Location, Description, Class, Updated, Mainteinance_Schedule)
				VALUES (%s, %s, %s, %s, %s, %s, %s);
			"""
			try:
				cursor.execute(insert_query, (sub['Name'], sub['Manufacturer'], sub['Location'], sub['Description'], sub['Class'], sub['Updated'], sub['Mainteinance_Schedule']))
			
			except pymysql.Error as e:
				print(f"Error: {e}")

		logger.info("cmdb_ci_endpoint_subnet : Successfully inserted data into table")
		connection.commit()
		connection.close()
	except Exception as e:
		logger.info("cmdb_ci_endpoint_subnet : Error in inserting data into database : %s ", str(e))
		servicenow_response(f"{const.EXECUTION_ERROR} : {str(e)}")

# Function to send API response to servicenow
def servicenow_response(response):
	url = 'https://servicecafedev.service-now.com/api/cmd/aws_etl_discovery/Response_from_AWS'
	username = 'AWS-IE-User'
	password = '+YDOEp7dJij!r8rv1'  
	payload = {
	"aws_response" : response
	}

	result = requests.post(url, auth=HTTPBasicAuth(username, password), json=payload)
	if result.status_code == 200:
		logger.info('%s : Successfully sent this response to ServiceNow.', response)
	else:
		logger.info('%s : Error in sending this response to ServiceNow.', response)

# Function to create master session on aws master account
def create_master_session():
	temporary_credentials = credentials
	session = boto3.Session(
		aws_access_key_id=temporary_credentials['AccessKeyId'],
		aws_secret_access_key=temporary_credentials['SecretAccessKey'],
		aws_session_token=temporary_credentials['SessionToken']
	)
	return session

# Function to check sql table in database
def check_table_in_database(table_name, cursor):
	current_date = datetime.now()
	current_time = datetime.now().strftime("%H:%M:%S")
	previous_date = (current_date - timedelta(days=1)).strftime("%d-%m-%Y")

	show_table = f"SHOW TABLES LIKE '{table_name}'"
	cursor.execute(show_table)
	tb = cursor.fetchone() 
	if tb:
		rename_table_query = f"ALTER TABLE `{table_name}` RENAME TO `{table_name}_{previous_date}_{current_time}`"
		cursor.execute(rename_table_query)

# Function to assume a temporary session on member accounts
def assume_new_session(master_session, role):
	sts = master_session.client('sts')
	assumed_role = sts.assume_role(
		RoleArn=role,
		RoleSessionName='SessionName'  
	)
	credentials = assumed_role['Credentials']
	assumed_session = boto3.Session(
		aws_access_key_id=credentials['AccessKeyId'],
		aws_secret_access_key=credentials['SecretAccessKey'],
		aws_session_token=credentials['SessionToken'],
		region_name='us-east-1' 
	)
	return assumed_session, credentials

handler()
