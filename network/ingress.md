### 租户/namespace ingress 隔离

- Running Multiple NGINX Ingress Controllers: https://docs.nginx.com/nginx-ingress-controller/installation/running-multiple-ingress-controllers/#running-multiple-nginx-ingress-controllers
- 问题:
  - 如果为不同ns部署不同的ingress-controller，不同的ingress-controller要么listen不同的ip(在不同节点上)，要么listen不同的port。(华为云的方案是在ingress 前面加了LB，通过LB的不同port区分)
  - work-around: 创建ingress时，全局过滤 host+path 保证唯一，防止冲突