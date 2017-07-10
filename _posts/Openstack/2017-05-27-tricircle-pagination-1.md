---
layout: post
#标题配置
title:  Tricircle pagination 分页问题(一)
#时间配置
date:   2017-05-27 10:49:00 +0800
#大类配置
categories: Tricircle
#小类配置
tag: Development notes
---

* content
{:toc}

# 1.任务
1. implement routing pagination
2. implemnet async job cli in python-tricircleclient

# 2.参考资料
read neutron pagination source code

refer to pod routing cli implemention in python-tricircleclient<br/>
[github-tricircleclient](https://github.com/openstack/python-tricircleclient)<br/>
some patches to implement pod routing cli<br/>
[review patches](https://review.openstack.org/#/q/project:openstack/python-tricircleclient)<br/>
developer guide pagination的参数<br/>
[pagination API 参数](https://developer.openstack.org/api-ref/networking/v2/#pagination)<br/>

# 3.Neutron中的分页实现
## 3.1 oslo.db底层paginate_query源码 
github地址[link](https://github.com/openstack/oslo.db/commit/dea700d13e4c152bf1a484b62cc2bce329ef6fa9#diff-7d84830d3af287b534a2f8a6d74e2de5R151)<br/>


# 4.工作log
## 4.1 参考code
**tricircle中的paginate_query**
```python
central_plugin.py

 def _get_ports_from_db_with_number(self, context,
                                       number, last_port_id, top_bottom_map,
                                       filters=None):
        query = context.session.query(models_v2.Port)
        # set step as two times of number to have better chance to obtain all
        # ports we need
        search_step = number * 2
        if search_step < 100:
            search_step = 100
        query = self._apply_ports_filters(query, models_v2.Port, filters)
        query = sa_utils.paginate_query(
            query, models_v2.Port, search_step,
            # create a dummy port object
            marker=models_v2.Port(
                id=last_port_id) if last_port_id else None,
            sort_keys=['id'],
            sort_dirs=['desc-nullsfirst'])
        total = 0
        ret = []
        for port in query:
            total += 1
            if port['id'] not in top_bottom_map:
                ret.append(self._make_port_dict(port))
            if len(ret) == number:
                return ret
        # NOTE(zhiyuan) we have traversed all the ports
        if total < search_step:
            return ret
        else:
            ret.extend(self._get_ports_from_db_with_number(
                context, number - len(ret), ret[-1]['id'], top_bottom_map))
            return ret

```
可以看出通过paginate_query查询的步骤为：
1. query语句的生成，包括基本query上添加filters的过滤条件组合成新的具体query
2. 通过paginate_query进行查询，设置limit，mark，sort
3. 对得到的查询result进行处理

**paginate_query中对filters的转换**
```python
@staticmethod
    def _apply_ports_filters(query, model, filters):
        if not filters:
            return query

        fixed_ips = filters.pop('fixed_ips', {})
        ip_addresses = fixed_ips.get('ip_address')
        subnet_ids = fixed_ips.get('subnet_id')
        if ip_addresses or subnet_ids:
            query = query.join(models_v2.Port.fixed_ips)
        if ip_addresses:
            query = query.filter(
                models_v2.IPAllocation.ip_address.in_(ip_addresses))
        if subnet_ids:
            query = query.filter(
                models_v2.IPAllocation.subnet_id.in_(subnet_ids))

        for key, value in six.iteritems(filters):
            column = getattr(model, key, None)
            if column is not None:
                if not value:
                    query = query.filter(sql.false())
                    return query
                query = query.filter(column.in_(value))
        return query
```
可以看到对于filter中的限制，通过query.filter(column.in_(value))进行条件的限制

**tricircle中原本的job查询逻辑**
```python
 @expose(generic=True, template='json')
    def get_all(self, **kwargs):
        """Get all the jobs. Using filters, only get a subset of jobs.

        :param kwargs: job filters
        :return: a list of jobs
        """
        context = t_context.extract_context_from_environ()

        if not policy.enforce(context, policy.ADMIN_API_JOB_LIST):
            return utils.format_api_error(
                403, _('Unauthorized to show all jobs'))

        is_valid_filter, filters = self._get_filters(kwargs)

        if not is_valid_filter:
            msg = (_('Unsupported filter type: %(filters)s') % {
                'filters': ', '.join([filter_name for filter_name in filters])
            })
            return utils.format_api_error(400, msg)

        filters = [{'key': key,
                    'comparator': 'eq',
                    'value': value} for key, value in six.iteritems(filters)]

        try:
            jobs_in_job_table = db_api.list_jobs(context, filters)
            jobs_in_job_log_table = db_api.list_jobs_from_log(context, filters)
            jobs = jobs_in_job_table + jobs_in_job_log_table
            return {'jobs': [self._get_more_readable_job(job) for job in jobs]}
        except Exception as e:
            LOG.exception('Failed to show all asynchronous jobs: '
                          '%(exception)s ', {'exception': e})
            return utils.format_api_error(
                500, _('Failed to show all asynchronous jobs'))
```
可以看到对于jobs的显示，query分别从async_jobs和async_job_logs两张表中进行查询，所以这里要用到union进行两个表的联合查询
## 4.2 tricircle中记录job信息的表结构

登录部署tricircle的机器，进行数据库中表的查看

    stack@stack-VirtualBox:~/devstack$ mysql -hlocalhost -uroot -p
    Enter password: 
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 5
    Server version: 5.7.18-0ubuntu0.16.04.1 (Ubuntu)
    
    Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.
    
    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.
    
    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | cinder             |
    | glance             |
    | keystone           |
    | mysql              |
    | neutron            |
    | neutron0           |
    | nova               |
    | nova_api           |
    | nova_cell0         |
    | performance_schema |
    | sys                |
    | tricircle          |
    +--------------------+
    13 rows in set (0.01 sec)
    
    mysql> use tricircle;
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A
    
    Database changed
    mysql> show tables;
    +---------------------+
    | Tables_in_tricircle |
    +---------------------+
    | async_job_logs      |
    | async_jobs          |
    | cached_endpoints    |
    | migrate_version     |
    | pods                |
    | resource_routings   |
    | shadow_agents       |
    +---------------------+
    7 rows in set (0.00 sec)
    mysql> select colmns from async_jobs;
    ERROR 1054 (42S22): Unknown column 'colmns' in 'field list'
    mysql> show colmns from async_jobs;
    ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'colmns from async_jobs' at line 1
    mysql> show columns from async_jobs;p
    +-------------+--------------+------+-----+-------------------+-------+
    | Field       | Type         | Null | Key | Default           | Extra |
    +-------------+--------------+------+-----+-------------------+-------+
    | id          | varchar(36)  | NO   | PRI | NULL              |       |
    | type        | varchar(36)  | YES  | MUL | NULL              |       |
    | timestamp   | timestamp    | YES  | MUL | CURRENT_TIMESTAMP |       |
    | status      | varchar(36)  | YES  |     | NULL              |       |
    | resource_id | varchar(127) | YES  |     | NULL              |       |
    | extra_id    | varchar(36)  | YES  |     | NULL              |       |
    | project_id  | varchar(36)  | YES  |     | NULL              |       |
    +-------------+--------------+------+-----+-------------------+-------+
    7 rows in set (0.00 sec)
    
        -> ;
    ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'p' at line 1
    mysql> show columns from async_job_logs;
    +-------------+--------------+------+-----+-------------------+-------+
    | Field       | Type         | Null | Key | Default           | Extra |
    +-------------+--------------+------+-----+-------------------+-------+
    | id          | varchar(36)  | NO   | PRI | NULL              |       |
    | resource_id | varchar(127) | YES  |     | NULL              |       |
    | type        | varchar(36)  | YES  |     | NULL              |       |
    | timestamp   | timestamp    | YES  | MUL | CURRENT_TIMESTAMP |       |
    | project_id  | varchar(36)  | YES  |     | NULL              |       |
    +-------------+--------------+------+-----+-------------------+-------+
    5 rows in set (0.00 sec)
    
    mysql> select * from async_jobs limit 5;
    Empty set (0.00 sec)
    
    mysql> select * from async_job_logs limit 5;
    +--------------------------------------+---------------------------------------------------------------------------+----------------+---------------------+----------------------------------+
    | id                                   | resource_id                                                               | type           | timestamp           | project_id                       |
    +--------------------------------------+---------------------------------------------------------------------------+----------------+---------------------+----------------------------------+
    | 22e3e24e-0e2e-4e76-b849-644cab4ef56b | 9edb408769274e568598a2686d21b98a                                          | seg_rule_setup | 2017-05-11 22:03:47 | 9edb408769274e568598a2686d21b98a |
    | 2d12d3ef-3a6d-412c-b86d-d31c6a2234a1 | 9edb408769274e568598a2686d21b98a                                          | seg_rule_setup | 2017-05-11 21:33:18 | 9edb408769274e568598a2686d21b98a |
    | 360fc298-9fd8-46ac-b3cb-4791bc527262 | 9edb408769274e568598a2686d21b98a                                          | seg_rule_setup | 2017-05-11 21:04:44 | 9edb408769274e568598a2686d21b98a |
    | 4fd383ad-5957-4160-9404-74d67e210f67 | 322f6920-f82a-427d-88dc-83d6f7d778f9#18e24fdf-d3be-40fc-9f6a-77f40e77b022 | port_delete    | 2017-05-11 21:28:58 | 9edb408769274e568598a2686d21b98a |
    | 6d059124-aa87-4354-a33f-c6d06bc750cb | 322f6920-f82a-427d-88dc-83d6f7d778f9#d5c02752-5a8b-43ca-996b-542e4ac46e07 | port_delete    | 2017-05-11 21:11:12 | 9edb408769274e568598a2686d21b98a |
    +--------------------------------------+---------------------------------------------------------------------------+----------------+---------------------+----------------------------------+
    5 rows in set (0.00 sec)
    
    mysql> select id, type, timestamp from async_jobs union select id, type, timestamp from async_job_logs;
    +--------------------------------------+----------------+---------------------+
    | id                                   | type           | timestamp           |
    +--------------------------------------+----------------+---------------------+
    | 22e3e24e-0e2e-4e76-b849-644cab4ef56b | seg_rule_setup | 2017-05-11 22:03:47 |
    | 2d12d3ef-3a6d-412c-b86d-d31c6a2234a1 | seg_rule_setup | 2017-05-11 21:33:18 |
    | 360fc298-9fd8-46ac-b3cb-4791bc527262 | seg_rule_setup | 2017-05-11 21:04:44 |
    | 4fd383ad-5957-4160-9404-74d67e210f67 | port_delete    | 2017-05-11 21:28:58 |
    | 6d059124-aa87-4354-a33f-c6d06bc750cb | port_delete    | 2017-05-11 21:11:12 |
    | 9bb65105-7d5f-403c-ac8d-77347dda4433 | port_delete    | 2017-05-11 20:43:38 |
    | 9e4f5ecc-8d60-45d4-8b79-5d5fc4fac201 | port_delete    | 2017-05-11 21:04:45 |
    | a616c73a-e93a-4823-baad-2f6d970de983 | port_delete    | 2017-05-11 21:09:19 |
    | ab9e7848-c7e8-4a53-9915-a487e103bed5 | seg_rule_setup | 2017-05-11 21:11:11 |
    | ba4d6ba6-b3db-4924-9b3b-059640dee567 | seg_rule_setup | 2017-05-11 21:09:18 |
    | bd1b6c5b-9e8b-49c8-8016-54e96112220c | seg_rule_setup | 2017-05-11 21:28:57 |
    | c1849f1a-a163-46b1-9420-415d812b6e29 | seg_rule_setup | 2017-05-11 20:43:37 |
    | c1ee4da2-92cb-482e-9ed7-fa209129f626 | port_delete    | 2017-05-11 21:55:40 |
    | c61f6a25-5144-427f-85d5-6913547a5ebd | port_delete    | 2017-05-11 21:33:19 |
    | d1644eb9-0bfd-408f-842a-4b011dc1b8d9 | port_delete    | 2017-05-11 22:03:47 |
    | e4b7df12-e197-4327-b546-9f53e22fb545 | seg_rule_setup | 2017-05-11 20:46:31 |
    | fb968073-18f6-405b-a571-794dd02b93f3 | seg_rule_setup | 2017-05-11 21:55:39 |
    +--------------------------------------+----------------+---------------------+
    17 rows in set (0.00 sec)
    
    mysql> select id, type, timestamp from async_jobs union select id, type, timestamp from async_job_logs limit 8;
    +--------------------------------------+----------------+---------------------+
    | id                                   | type           | timestamp           |
    +--------------------------------------+----------------+---------------------+
    | 22e3e24e-0e2e-4e76-b849-644cab4ef56b | seg_rule_setup | 2017-05-11 22:03:47 |
    | 2d12d3ef-3a6d-412c-b86d-d31c6a2234a1 | seg_rule_setup | 2017-05-11 21:33:18 |
    | 360fc298-9fd8-46ac-b3cb-4791bc527262 | seg_rule_setup | 2017-05-11 21:04:44 |
    | 4fd383ad-5957-4160-9404-74d67e210f67 | port_delete    | 2017-05-11 21:28:58 |
    | 6d059124-aa87-4354-a33f-c6d06bc750cb | port_delete    | 2017-05-11 21:11:12 |
    | 9bb65105-7d5f-403c-ac8d-77347dda4433 | port_delete    | 2017-05-11 20:43:38 |
    | 9e4f5ecc-8d60-45d4-8b79-5d5fc4fac201 | port_delete    | 2017-05-11 21:04:45 |
    | a616c73a-e93a-4823-baad-2f6d970de983 | port_delete    | 2017-05-11 21:09:19 |
    +--------------------------------------+----------------+---------------------+
    8 rows in set (0.00 sec)
    
    mysql> select * from async_jobs union select * from async_job_logs limit 8;
    ERROR 1222 (21000): The used SELECT statements have a different number of columns
    mysql> 
    
通过对表结构的查询可以看出，对于union操作两张表，只能选取到的columns为：</br>
id, type, timestamp, resource_id, project_id 这5个columns

## 4.3 python中sqlalchemy的简单理解和union使用
项目中的sqlalchemy的参考操作语句

     def _select_dhcp_ips_for_network_ids(self, context, network_ids):
            if not network_ids:
                return {}
            query = context.session.query(models_v2.Port.mac_address,
                                          models_v2.Port.network_id,
                                          models_v2.IPAllocation.ip_address)
            query = query.join(models_v2.IPAllocation)
            query = query.filter(models_v2.Port.network_id.in_(network_ids))
            owner = const.DEVICE_OWNER_DHCP
            query = query.filter(models_v2.Port.device_owner == owner)
            ips = {}
    
            for network_id in network_ids:
                ips[network_id] = []
    
            for mac_address, network_id, ip in query:
                if (netaddr.IPAddress(ip).version == 6
                    and not netaddr.IPAddress(ip).is_link_local()):
                    ip = str(netutils.get_ipv6_addr_by_EUI64(const.IPv6_LLA_PREFIX,
                        mac_address))
                if ip not in ips[network_id]:
                    ips[network_id].append(ip)
    
            return ips

## 4.4 mycode

```python
tricircl/api/controllers/job.py

# Licensed under the Apache License, Version 2.0 (the "License"); you may
# not use this file except in compliance with the License. You may obtain
# a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
# WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
# License for the specific language governing permissions and limitations
# under the License.

import pecan
from pecan import expose
from pecan import rest
import six

from oslo_log import log as logging
from oslo_utils import timeutils
from oslo_db.sqlalchemy import utils as sa_utils

from sqlalchemy import sql

from tricircle.common import constants
import tricircle.common.context as t_context
import tricircle.common.exceptions as t_exc
from tricircle.common.i18n import _
from tricircle.common import policy
from tricircle.common import utils
from tricircle.common import xrpcapi
from tricircle.db import api as db_api

from tricircle.db import models

LOG = logging.getLogger(__name__)


class AsyncJobController(rest.RestController):
    # with AsyncJobController, admin can create, show, delete and
    # redo asynchronous jobs

    def __init__(self):
        self.xjob_handler = xrpcapi.XJobAPI()

    @expose(generic=True, template='json')
    def post(self, **kw):
        context = t_context.extract_context_from_environ()
        job_resource_map = constants.job_resource_map

        if not policy.enforce(context, policy.ADMIN_API_JOB_CREATE):
            return utils.format_api_error(
                403, _("Unauthorized to create a job"))

        if 'job' not in kw:
            return utils.format_api_error(
                400, _("Request body not found"))

        job = kw['job']

        for field in ('type', 'project_id'):
            value = job.get(field)
            if value is None:
                return utils.format_api_error(
                    400, _("%(field)s isn't provided in request body") % {
                        'field': field})
            elif len(value.strip()) == 0:
                return utils.format_api_error(
                    400, _("%(field)s can't be empty") % {'field': field})

        if job['type'] not in job_resource_map.keys():
            return utils.format_api_error(
                400, _('There is no such job type: %(job_type)s') % {
                    'job_type': job['type']})

        job_type = job['type']
        project_id = job['project_id']

        if 'resource' not in job:
            return utils.format_api_error(
                400, _('Failed to create job, because the resource is not'
                       ' specified'))

        # verify that all given resources are exactly needed
        request_fields = set(job['resource'].keys())
        require_fields = set([resource_id
                              for resource_type, resource_id in
                              job_resource_map[job_type]])
        missing_fields = require_fields - request_fields
        redundant_fields = request_fields - require_fields

        if missing_fields:
                return utils.format_api_error(
                    400, _('Some required fields are not specified:'
                           ' %(field)s') % {'field': missing_fields})
        if redundant_fields:
                return utils.format_api_error(
                    400, _('Some fields are redundant: %(field)s') % {
                        'field': redundant_fields})

        # validate whether the project id is legal
        resource_type_1, resource_id_1 = (
            constants.job_primary_resource_map[job_type])
        if resource_type_1 is not None:
            filter = [{'key': 'project_id', 'comparator': 'eq',
                       'value': project_id},
                      {'key': 'resource_type', 'comparator': 'eq',
                       'value': resource_type_1},
                      {'key': 'top_id', 'comparator': 'eq',
                       'value': job['resource'][resource_id_1]}]

            routings = db_api.list_resource_routings(context, filter)
            if not routings:
                msg = (_("%(resource)s %(resource_id)s doesn't belong to the"
                         " project %(project_id)s") %
                       {'resource': resource_type_1,
                        'resource_id': job['resource'][resource_id_1],
                        'project_id': project_id})
                return utils.format_api_error(400, msg)

        # if job_type = seg_rule_setup, we should ensure the project id
        # is equal to the one from resource.
        if job_type == constants.JT_SEG_RULE_SETUP:
            if job['project_id'] != job['resource']['project_id']:
                msg = (_("Specified project_id %(project_id_1)s and resource's"
                         " project_id %(project_id_2)s are different") %
                       {'project_id_1': job['project_id'],
                        'project_id_2': job['resource']['project_id']})
                return utils.format_api_error(400, msg)

        # combine uuid into target resource id
        resource_id = '#'.join([job['resource'][resource_id]
                                for resource_type, resource_id
                                in job_resource_map[job_type]])

        try:
            # create a job and put it into execution immediately
            self.xjob_handler.invoke_method(context, project_id,
                                            constants.job_handles[job_type],
                                            job_type, resource_id)
        except Exception as e:
            LOG.exception('Failed to create job: '
                          '%(exception)s ', {'exception': e})
            return utils.format_api_error(
                500, _('Failed to create a job'))

        new_job = db_api.get_latest_job(context, constants.JS_New, job_type,
                                        resource_id)
        return {'job': self._get_more_readable_job(new_job)}

    @expose(generic=True, template='json')
    def get_one(self, id, **kwargs):
        """the return value may vary according to the value of id

        :param id: 1) if id = 'schemas', return job schemas
                   2) if id = 'detail', return all jobs
                   3) if id = $job_id, return detailed single job info
        :return: return value is decided by id parameter
        """
        context = t_context.extract_context_from_environ()
        job_resource_map = constants.job_resource_map

        if not policy.enforce(context, policy.ADMIN_API_JOB_SCHEMA_LIST):
            return utils.format_api_error(
                403, _('Unauthorized to show job information'))

        if id == 'schemas':
            job_schemas = []
            for job_type in job_resource_map.keys():
                job = {}
                resource = []
                for resource_type, resource_id in job_resource_map[job_type]:
                    resource.append(resource_id)

                job['resource'] = resource
                job['type'] = job_type
                job_schemas.append(job)

            return {'schemas': job_schemas}

        if id == 'detail':
            return self.get_all(**kwargs)

        try:
            job = db_api.get_job(context, id)
            return {'job': self._get_more_readable_job(job)}
        except Exception:
            try:
                job = db_api.get_job_from_log(context, id)
                return {'job': self._get_more_readable_job(job)}
            except t_exc.ResourceNotFound:
                return utils.format_api_error(
                    404, _('Resource not found'))

    @staticmethod
    def _apply_asyncJob_filters(query_jobs, query_logs, model_jobs, model_logs, filters):
        """union async_job_logs and filter the query
        :return: query
        """
        if not filters:
            query = query_jobs.union(query_logs)
            return query

        for key, value in six.iteritems(filters):
            column = getattr(model_jobs, key, None)
            if column is not None:
                if not value:
                    query_jobs = query_jobs.filter(sql.false())
                    break
                query_jobs = query_jobs.filter(column.in_(value))

        for key, value in six.iteritems(filters):
            column = getattr(model_logs, key, None)
            if column is not None:
                if not value:
                    query_logs = query_logs.filter(sql.false())
                    break
                query_logs = query_logs.filter(column.in_(value))

        query = query_jobs.union(query_logs)

        return query

    @expose(generic=True, template='json')
    def get_all(self, number=0, last_job_id, **kwargs):
        """Get all the jobs. Using filters, only get a subset of jobs.

        :param kwargs: job filters
        :return: a list of jobs
        """
        context = t_context.extract_context_from_environ()

        if not policy.enforce(context, policy.ADMIN_API_JOB_LIST):
            return utils.format_api_error(
                403, _('Unauthorized to show all jobs'))

        is_valid_filter, filters = self._get_filters(kwargs)

        if not is_valid_filter:
            msg = (_('Unsupported filter type: %(filters)s') % {
                'filters': ', '.join([filter_name for filter_name in filters])
            })
            return utils.format_api_error(400, msg)

        filters = [{'key': key,
                    'comparator': 'eq',
                    'value': value} for key, value in six.iteritems(filters)]

        # create query
        query_jobs = context.session.query(models.AsyncJob.id,
                                      models.AsyncJob.type,
                                      models.AsyncJob.timestamp,
                                      models.AsyncJob.resource_id,
                                      models.AsyncJob.project_id)
        query_logs = context.session.query(models.AsyncJobLog.id,
                                           models.AsyncJobLog.type,
                                           models.AsyncJobLog.timestamp,
                                           models.AsyncJobLog.resource_id,
                                           models.AsyncJobLog.project_id)

        query = self._apply_asyncJob_filters(query_jobs, query_logs, models.AsyncJob,models.AsyncJobLog, filters)

        search_step = 100 if number<=0 or number>100 else number

        try:
            jobs = sa_utils.paginate_query(
                query, models.AsyncJob, search_step,
                marker=models.AsyncJob(
                    id=last_job_id) if last_job_id else None,
                sort_keys=['id'],
                sort_dirs=['desc-nullsfirst'])
        except Exception as e:
            LOG.exception('Failed to show all asynchronous jobs: '
                          '%(exception)s ', {'exception': e})
            return utils.format_api_error(
                500, _('Failed to show all asynchronous jobs'))

        # deal query result
        return {'jobs': [self._get_more_readable_job(job) for job in jobs]}

    # make the job status and resource id more human readable. Split
    # resource id into several member uuid(s) to provide more detailed resource
    # information. If job entry is from job table, then remove resource id
    # and extra id from job attributes. If job entry is from job log table,
    # only remove resource id from job attributes.
    def _get_more_readable_job(self, job):
        job_resource_map = constants.job_resource_map

        if 'status' in job:
            job['status'] = constants.job_status_map[job['status']]
        else:
            job['status'] = constants.job_status_map[constants.JS_Success]

        job['resource'] = dict(zip([resource_id
                                    for resource_type, resource_id
                                    in job_resource_map[job['type']]],
                                   job['resource_id'].split('#')))
        job.pop('resource_id')

        if "extra_id" in job:
            job.pop('extra_id')

        return job

    def _get_filters(self, params):
        """Return a dictionary of query param filters from the request.

        :param params: the URI params coming from the wsgi layer
        :return (flag, filters), flag indicates whether the filters are valid,
        and the filters denote a list of key-value pairs.
        """
        filters = {}
        unsupported_filters = {}
        for filter_name in params:
            if filter_name in constants.JOB_LIST_SUPPORTED_FILTERS:
                # map filter name
                if filter_name == 'status':
                    job_status_in_db = self._get_job_status_in_db(
                        params.get(filter_name))
                    filters[filter_name] = job_status_in_db
                    continue
                filters[filter_name] = params.get(filter_name)
            else:
                unsupported_filters[filter_name] = params.get(filter_name)

        if unsupported_filters:
            return False, unsupported_filters
        return True, filters

    # map user input job status to job status stored in database
    def _get_job_status_in_db(self, job_status):
        job_status_map = {
            'fail': constants.JS_Fail,
            'success': constants.JS_Success,
            'running': constants.JS_Running,
            'new': constants.JS_New
        }
        job_status_lower = job_status.lower()
        if job_status_lower in job_status_map:
            return job_status_map[job_status_lower]
        return job_status

    @expose(generic=True, template='json')
    def delete(self, job_id):
        # delete a job from the database. If the job is running, the delete
        # operation will fail. In other cases, job will be deleted directly.
        context = t_context.extract_context_from_environ()

        if not policy.enforce(context, policy.ADMIN_API_JOB_DELETE):
            return utils.format_api_error(
                403, _('Unauthorized to delete a job'))

        try:
            db_api.get_job_from_log(context, job_id)
            return utils.format_api_error(
                400, _('Job %(job_id)s is from job log') % {'job_id': job_id})
        except Exception:
            try:
                job = db_api.get_job(context, job_id)
            except t_exc.ResourceNotFound:
                return utils.format_api_error(
                    404, _('Job %(job_id)s not found') % {'job_id': job_id})
        try:
            # if job status = RUNNING, notify user this new one, delete
            # operation fails.
            if job['status'] == constants.JS_Running:
                return utils.format_api_error(
                    400, (_('Failed to delete the running job %(job_id)s') %
                          {"job_id": job_id}))
            # if job status = SUCCESS, move the job entry to job log table,
            # then delete it from job table.
            elif job['status'] == constants.JS_Success:
                db_api.finish_job(context, job_id, True, timeutils.utcnow())
                pecan.response.status = 200
                return {}

            db_api.delete_job(context, job_id)
            pecan.response.status = 200
            return {}
        except Exception as e:
            LOG.exception('Failed to delete the job: '
                          '%(exception)s ', {'exception': e})
            return utils.format_api_error(
                500, _('Failed to delete the job'))

    @expose(generic=True, template='json')
    def put(self, job_id):
        # we use HTTP/HTTPS PUT method to redo a job. Regularly PUT method
        # requires a request body, but considering the job redo operation
        # doesn't need more information other than job id, we will issue
        # this request without a request body.
        context = t_context.extract_context_from_environ()

        if not policy.enforce(context, policy.ADMIN_API_JOB_REDO):
            return utils.format_api_error(
                403, _('Unauthorized to redo a job'))

        try:
            db_api.get_job_from_log(context, job_id)
            return utils.format_api_error(
                400, _('Job %(job_id)s is from job log') % {'job_id': job_id})
        except Exception:
            try:
                job = db_api.get_job(context, job_id)
            except t_exc.ResourceNotFound:
                return utils.format_api_error(
                    404, _('Job %(job_id)s not found') % {'job_id': job_id})

        try:
            # if status = RUNNING, notify user this new one and then exit
            if job['status'] == constants.JS_Running:
                return utils.format_api_error(
                    400, (_("Can't redo job %(job_id)s which is running") %
                          {'job_id': job['id']}))
            # if status = SUCCESS, notify user this new one and then exit
            elif job['status'] == constants.JS_Success:
                msg = (_("Can't redo job %(job_id)s which had run successfully"
                         ) % {'job_id': job['id']})
                return utils.format_api_error(400, msg)
            # if job status =  FAIL or job status = NEW, redo it immediately
            self.xjob_handler.invoke_method(context, job['project_id'],
                                            constants.job_handles[job['type']],
                                            job['type'], job['resource_id'])
        except Exception as e:
            LOG.exception('Failed to redo the job: '
                          '%(exception)s ', {'exception': e})
            return utils.format_api_error(
                500, _('Failed to redo the job'))

```