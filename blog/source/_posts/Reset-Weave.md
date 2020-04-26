---
title: Reset Weave
date: 2020-04-01 20:03:08
tags:
categories:
	- Kubernetes
---



```

sudo curl -L git.io/weave -o /usr/local/bin/weave

sudo chmod a+x /usr/local/bin/weave
 
sudo weave reset --force

```

如果需要删除机器上的文件

```
## Clean weave binaries
sudo rm /opt/cni/bin/weave-*
```






### Reference

https://gist.github.com/carlosedp/5040f4a1b2c97c1fa260a3409b5f14f9