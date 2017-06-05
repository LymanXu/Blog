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

# 任务
1. implement routing pagination
2. implemnet async job cli in python-tricircleclient

# 参考资料
read neutron pagination source code

refer to pod routing cli implemention in python-tricircleclient<br/>
[github-tricircleclient](https://github.com/openstack/python-tricircleclient)<br/>
some patches to implement pod routing cli<br/>
[review patches](https://review.openstack.org/#/q/project:openstack/python-tricircleclient)<br/>
developer guide pagination的参数<br/>
[pagination API 参数](https://developer.openstack.org/api-ref/networking/v2/#pagination)<br/>

# Neutron中的分页实现
## oslo.db底层paginate_query源码 
github地址[link](https://github.com/openstack/oslo.db/commit/dea700d13e4c152bf1a484b62cc2bce329ef6fa9#diff-7d84830d3af287b534a2f8a6d74e2de5R151)<br/>


# 工作log
## 参考code
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

## 技术实现

