---
title: "Tungstenfabric CNI源码 -- NetworkPolicy"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - k8s
  - tungstenfabric
  - cni
  - networkpolicy

---


## 背景
tungstenfabric cni通过watch kubernetes apiserver中指定的资源，并在sdn中创建对应的网络设备来实现对应功能。本文重点介绍cni针对networkpolicy的处理，根据源码逐步分析。

## 架构&流程图

![arch_network_policy](http://hujin.dynv6.net:50008/index.php?user/publicLink&fid=826f91FDc9_ivLXP-2TgqvVlTnWAWUUm5hE6qr_ef-KVP_3RWxuNClqp0L86chmPdHcV9GGK4fY5BLr3ualYY8IkFXQI_OHUoI2btmUla-PRxN494bUH8DsY7FAajC-fCOHFQHOBwYVnU8BSRZOxaVGZsp2WPTQ&file_name=/arch_network_policy.png)


## 源码解析

在process方法中会处理networkpolicy的创建、更新和删除。这里我们先看下创建和更新

创建和更新方法中有两个步骤：

- _add_labels： 获取networkpolicy中selector相关的lable，并在sdn中创建或更新对应的tag资源
- vnc_network_policy_add：在sdn中创建对应的aps（application policy set）policy资源


在_add_labels中：
    
    def _get_np_pod_selector(self, spec):
        pod_selector = spec.get('podSelector')
        if not pod_selector or 'matchLabels' not in pod_selector:
            labels = {}
        else:
            labels = pod_selector.get('matchLabels')
        return labels
    
    def _add_labels(self, event, namespace, np_uuid):
        all_labels = []
        spec = event['object']['spec']
        if spec:
            # Get pod selector labels.
            all_labels.append(self._get_np_pod_selector(spec))

            # Get ingress podSelector labels
            ingress_spec_list = spec.get("ingress", [])
            for ingress_spec in ingress_spec_list:
                from_rules = ingress_spec.get('from', [])
                for from_rule in from_rules:
                    if 'namespaceSelector' in from_rule:
                        all_labels.append(
                            from_rule.get('namespaceSelector').get(
                                'matchLabels', {}))
                    if 'podSelector' in from_rule:
                        all_labels.append(
                            from_rule.get('podSelector').get('matchLabels', {}))

            # Call label mgmt API.
            self._labels.process(np_uuid, list_curr_labels_dict=all_labels)

这里可以看到程序从networkpolicy的spec中获取了podSelector，从ingress中获取了namespaceSelector和podSelector对应的labels，最终在labels资源的process方法中进行处理

我们在label_cache.py的process中可以看到：

    def process(self, obj_uuid, curr_labels={}, list_curr_labels_dict=[]):
        ...
        all_labels = set()

        if list_curr_labels_dict:
            for labels_dict in list_curr_labels_dict:
                for key, value in labels_dict.items():
                    key, value = self._validate_key_value(key, value)
                    # Construct the label key.
                    label_key = self._update_label_to_guid_cache(key, value, obj_uuid)
                    # Construct a set of all input label keys.
                    all_labels.add(label_key)
        ...
针对从networkpolicy中传入的labels，在这里做了validate，然后做了_update_label_to_guid_cache：

    def _update_label_to_guid_cache(self, key, value, obj_uuid):

        # Construct the label key.
        label_key = self.get_key(key, value)

        # If an entry exists for this label, add guid to the existing entry.
        # If not, create one.
        ltg_cache = XLabelCache.k8s_label_to_guid_cache[self.resource_type]
        if label_key in ltg_cache:
            ltg_cache[label_key].add(obj_uuid)
        else:
            ltg_cache[label_key] = {obj_uuid}
            XLabelCache.label_add_cb(key, value)

        return label_key
        
这里将key和value先转换成sdn的tag格式，再调用XLabelCache.label_add_cb方法处理，这里的label_add_cb方法处理实际是一个callback方法，是在初始化时传入的，具体看下：

vnc_kubernetes.py中初始化VncKubernetes时：

    def __init_():
        ...
        # Register label add and delete callbacks with label management entity.
        label_cache.XLabelCache.register_label_add_callback(VncKubernetes.create_tags)
        label_cache.XLabelCache.register_label_delete_callback(VncKubernetes.delete_tags)

label_cache.py中

    @classmethod
    def register_label_add_callback(cls, cb_func):
        cls.label_add_cb = cb_func
        
    @classmethod
    def register_label_delete_callback(cls, cb_func):
        cls.label_delete_cb = cb_func
        
这里在初始化VncKubernetes是，调用label_cache.py中的register_label_add_callback，注册了两个方法，分别是创建和删除tag

我们在vnc_tag.py最终找到实际调用vnc创建和删除tag的方法：
    
    def create(self, type, value):
        tag_name = "=".join([type, value])
        tag = Tag(name=tag_name,
                  parent_obj=self.proj_obj,
                  tag_type_name=type,
                  tag_value=value)
        try:
            TagKM.add_annotations(self, tag, "default", tag_name)
            self._vnc_lib.tag_create(tag)
        except RefsExistError:
            # Tags cannot be updated.
            pass

        try:
            tag_obj = self._vnc_lib.tag_read(fq_name=tag.get_fq_name())
        except NoIdError as e:
            self._logger.error(
                "Unable to create tag [%s]. Error [%s]" %
                (tag.get_fq_name(), str(e)))
            return
        # Cache the object in local db.
        TagKM.locate(tag_obj.uuid)

    def delete(self, type, value):
        tag_uuid = TagKM.get_fq_name_to_uuid(
            self._construct_tag_fq_name(type, value))
        try:
            self._vnc_lib.tag_delete(id=tag_uuid)

            TagKM.delete(tag_uuid)
            self._logger.debug("Tag (%s) deleted successfully."
                               % (self._construct_tag_fq_name(type, value)))
        except RefsExistError:
            self._logger.debug("Tag (%s) deletion failed. Tag is in use."
                               % (self._construct_tag_fq_name(type, value)))
        except NoIdError:
            self._logger.debug("Tag delete failed. Tag [%s] not found."
                               % (self._construct_tag_fq_name(type, value)))

        return
        
这里我们可以得到一个结论，当存在多个k8s集群的时候，实际tag是共享的。也就是说当多个k8s集群有同名的labels时，实际在sdn中是复用的

看完labels的操作后，我们看下networkpolicy在sdn中的处理吧，在看之前我们需要知道一个前提条件：
    
vnc_kubernetes.py中：

    def _provision_cluster(self):
        ...
        # Create application policy set for the cluster project.
        VncSecurityPolicy.create_application_policy_set(
            vnc_kube_config.application_policy_set_name(), namespace=proj_obj.name)

我们发现kube-manager实际会为每个接入tungstenfabric的k8s集群创建一个aps，也就是一个防火墙。然后初始化三个policy，分别是：denyall/allowall/ingress，然后创建一些初始化规则，这里不展开讲了。

了解这个前提后，我们看下面的代码就容易理解了：

    def vnc_network_policy_add(self, event, namespace, name, uid):
        spec = event['object']['spec']
        if not spec:
            self._logger.error(
                "%s - %s:%s Spec Not Found"
                % (self._name, name, uid))
            return

        fw_policy_uuid = VncSecurityPolicy.create_firewall_policy(name, namespace,
                                                                  spec, k8s_uuid=uid)
        VncSecurityPolicy.add_firewall_policy(fw_policy_uuid)

        # Update kube config db entry for the network policy.
        np = NetworkPolicyKM.find_by_name_or_uuid(uid)
        if np:
            fw_policy_obj = self._vnc_lib.firewall_policy_read(id=fw_policy_uuid)
            np.set_vnc_fq_name(":".join(fw_policy_obj.get_fq_name()))
            
实际k8s中的networkpolicy对应sdn的资源就是aps policy资源。上面可以看到会在create_firewall_policy中创建一个policy，然后将policy绑定到aps中，也就是add_firewall_policy的动作。这里我们重点看下create_firewall_policy：

    @classmethod
    def create_firewall_policy(cls, name, namespace, spec, tag_last=False,
                               tag_after_tail=False, is_global=False,
                               k8s_uuid=None):
        ...
        policy_name = cls.get_firewall_policy_name(name, namespace, is_global)
        fw_policy_obj = FirewallPolicy(policy_name, pm_obj)

        custom_ann_kwargs = {}
        custom_ann_kwargs['k8s_uuid'] = k8s_uuid
        curr_fw_policy = None
        fw_rules_del_candidates = set()

        # If this firewall policy already exists, get its uuid.
        fw_policy_uuid = VncSecurityPolicy.get_firewall_policy_uuid(
            name, namespace, is_global)
        ...

        # Parse input spec and construct the list of rules for this FW policy.
        fw_rules = []
        deny_all_rule_uuid = None
        egress_deny_all_rule_uuid = None

        if spec is not None:
            fw_rules, deny_all_rule_uuid, egress_deny_all_rule_uuid =\
                FWRule.parser(name, namespace, pm_obj, spec)

        for rule in fw_rules:
            try:
                FirewallRuleKM.add_annotations(cls, rule, namespace, rule.name)
                rule_uuid = cls.vnc_lib.firewall_rule_create(rule)
            except RefsExistError:
                cls.vnc_lib.firewall_rule_update(rule)
                rule_uuid = rule.get_uuid()

                # The rule is in use and needs to stay.
                # Remove it from delete candidate collection.
                if fw_rules_del_candidates and\
                   rule_uuid in fw_rules_del_candidates:
                    fw_rules_del_candidates.remove(rule_uuid)

            rule_obj = cls.vnc_lib.firewall_rule_read(id=rule_uuid)
            FirewallRuleKM.locate(rule_uuid)

            fw_policy_obj.add_firewall_rule(
                rule_obj,
                cls.construct_sequence_number(fw_rules.index(rule)))

        if deny_all_rule_uuid:
            VncSecurityPolicy.add_firewall_rule(
                VncSecurityPolicy.deny_all_fw_policy_uuid, deny_all_rule_uuid)
            custom_ann_kwargs['deny_all_rule_uuid'] = deny_all_rule_uuid

        if egress_deny_all_rule_uuid:
            VncSecurityPolicy.add_firewall_rule(
                VncSecurityPolicy.deny_all_fw_policy_uuid,
                egress_deny_all_rule_uuid)
            custom_ann_kwargs['egress_deny_all_rule_uuid'] =\
                egress_deny_all_rule_uuid

        FirewallPolicyKM.add_annotations(
            VncSecurityPolicy.vnc_security_policy_instance,
            fw_policy_obj, namespace, name, None, **custom_ann_kwargs)

        try:
            fw_policy_uuid = cls.vnc_lib.firewall_policy_create(fw_policy_obj)
        except RefsExistError:
        ...
        
这里是创建aps policy，并提取spec中的数据生成policy rule并创建，然后将rule绑定到policy中，这里需要重点看下FWRule.parser，看看如何转化policy rule的：
    
    @classmethod
    def parser(cls, name, namespace, pobj, spec):

        fw_rules = []

        # Get pod selectors.
        podSelector_dict = cls._get_np_pod_selector(spec, namespace)
        tags = VncSecurityPolicy.get_tags_fn(podSelector_dict, True)

        deny_all_rule_uuid = None
        egress_deny_all_rule_uuid = None
        policy_types = spec.get('policyTypes', ['Ingress'])
        for policy_type in policy_types:
            if policy_type == 'Ingress':
                # Get ingress spec.
                ingress_spec_list = spec.get("ingress", [])
                for ingress_spec in ingress_spec_list:
                    fw_rules +=\
                        cls.ingress_parser(
                            name, namespace, pobj, tags,
                            ingress_spec, ingress_spec_list.index(ingress_spec))

                # Add ingress deny-all for all other non-explicit traffic.
                deny_all_rule_name = namespace + "-ingress-" + name + "-denyall"
                deny_all_rule_uuid =\
                    VncSecurityPolicy.create_firewall_rule_deny_all(
                        deny_all_rule_name, tags, namespace)

            if policy_type == 'Egress':
                # Get egress spec.
                egress_spec_list = spec.get("egress", [])
                for egress_spec in egress_spec_list:
                    fw_rules +=\
                        cls.egress_parser(name, namespace, pobj, tags,
                                          egress_spec)
                # Add egress deny-all for all other non-explicit traffic.
                egress_deny_all_rule_uuid =\
                    VncSecurityPolicy.create_firewall_rule_egress_deny_all(
                        name, namespace, tags)

        return fw_rules, deny_all_rule_uuid, egress_deny_all_rule_uuid
        
_get_np_pod_selector会获取spec中podSelector和namespace两个label的数据，然后通过get_tags_fn查询sdn中已经创建的tag数据，这里的tag数据有：

- podselector: matchLabels: xxx:xxx
- namespace:xxx

然后会获取spec中ingress和egress对应的规则，通过ingress_parser和egress_parser做转换，转换成具体的sdn policy rule格式，这里需要注意：

- 会优先根据spec中ingress和egress添加的规则创建policy rule
- ingress末尾会添加一条默认规则，一条ingress deny规则到此集群对应的deny-all的policy中
- 如果指定了egress规则，会在末尾添加一条deny规则到此集群的deny-all的policy中，即绑定此tag的资源无法访问任意其他资源


至此，我们基本分析完了networkpolicy的工作流程，但是我们似乎漏了点什么，policy和rule都创建了，但是如何生效的？ 资源和tag的绑定关系发生在什么时候？这里我们通过pod的创建流程来分析下

在pod创建过程中，我们看到有涉及labels的处理流程(vnc_pod.py process):

    def process(self, event):
        ...
        # Add implicit namespace labels on this pod.
        labels.update(self._get_namespace_labels(pod_namespace))
        self._labels.process(pod_id, labels)
和之前的流程类似，这里会获取pod中metadata的label，并创建出对应的tag资源

    def vnc_pod_add(self, pod_id, pod_name, pod_namespace, pod_node, node_ip,
                    labels, vm_vmi, fixed_ip=None, annotations_bandwidth_str=None):
        vm = VirtualMachineKM.get(pod_id)
        if vm:
            vm.pod_namespace = pod_namespace
            if not vm.virtual_router:
                self._link_vm_to_node(vm, pod_node, node_ip)
            self._set_label_to_pod_cache(labels, vm)

            # Update tags.
            self._set_tags_on_pod_vmi(pod_id)

            return vm
我们在创建pod的流程中发现，_set_tags_on_pod_vmi会将tag绑定到具体的资源中

    def _set_tags_on_pod_vmi(self, pod_id, old_lables=None):
        vmi_obj_list = []
        vm = VirtualMachineKM.get(pod_id)
        if vm:
            for vmi_id in list(vm.virtual_machine_interfaces):
                vmi_obj_list.append(
                    self._vnc_lib.virtual_machine_interface_read(id=vmi_id))

        for vmi_obj in vmi_obj_list:
            labels = self._labels.get_labels_dict(pod_id)
            self._vnc_lib.set_tags(vmi_obj, labels)
            if old_lables:
                diff_labels = {k:old_lables[k] for k in old_lables if k not in labels.keys()}
                for k, v in diff_labels.items():
                    self._vnc_lib.unset_tag(vmi_obj, k)
                    
代码中先从VirtualMachineKM中查询出vm对象，此时vm对象已经在上面的_set_label_to_pod_cache方法中将laables设置进去了；
先从数据库中查询出对应的vmi列表，然后依次调用vnc的set_tags方法，将tag和vmi进行绑定

这样整个流程就完成了。创建sdn资源的时候，创建tag并绑定到资源中；创建networkpolicy时，管理绑定不同tag的资源的行为