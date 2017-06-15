---
layout: post
#标题配置
title:  Tricircle pagination 分页问题
#时间配置
date:   2017-05-27 10:49:00 +0800
#大类配置
categories: Tricircle
#小类配置
tag: 技术文档
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

    stack@stack-VirtualBox:~/devstack$ <font color=#FF0000>mysql -hlocalhost -uroot -p</font>
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