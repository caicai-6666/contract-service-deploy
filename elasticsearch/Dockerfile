ARG ELASTICSEARCH_VERSION=9.4.5
FROM docker.elastic.co/elasticsearch/elasticsearch:${ELASTICSEARCH_VERSION}
# 插件随 ES 镜像构建，不复用跨版本插件卷，也不在每次启动时下载。
RUN bin/elasticsearch-plugin install --batch analysis-smartcn
