# Confluence

Confluence是专业的企业知识管理与协同工具，可用于构建企业wiki。其强大的编辑和站点管理特征能够帮助团队成员之间共享信息、文档协作、集体讨论、信息推送。

## 介绍

此chart使用Helm包管理器，在kubernetes集群上部署单节点confluence。

## 先决条件

- Kubernetes 1.6+ 
- PV provisioner support in the underlying infrastructure
- Postgresql 9.5
- helm3 +

## 安装Chart

部署：

```bash
$ helm install confluence ./confluence
```

或者，可以在安装chart时提供指定values.yaml文件：

```bash
$ helm install my-release -f values.yaml ./confluence
```

> **Tip**: 使用 `helm list`列出所有release。

## 卸载Chart

卸载：

```bash
$ helm uninstall confluence
```
