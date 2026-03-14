FROM eclipse-temurin:8-jre-jammy

# 手动添加本地 Tomcat 包
COPY tomcat/apache-tomcat-9.0.82.tar.gz /tmp/tomcat.tar.gz

# 安装 Tomcat
ENV CATALINA_HOME=/opt/tomcat
ENV PATH=$CATALINA_HOME/bin:$PATH
RUN mkdir -p "$CATALINA_HOME" && \
    tar xzvf /tmp/tomcat.tar.gz -C /opt/tomcat --strip-components=1 && \
    rm -rf /tmp/tomcat.tar.gz /opt/tomcat/webapps/*

# 替换为修复后的 server.xml
COPY conf/server.xml $CATALINA_HOME/conf/
COPY conf/context.xml $CATALINA_HOME/conf/

# 部署 WAR 包
COPY war/ruoyi-admin-docker.war $CATALINA_HOME/webapps/ROOT.war

# 生产级 JVM 参数
ENV JAVA_OPTS="-Xms512m -Xmx2048m -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8"

EXPOSE 8080
CMD ["catalina.sh", "run"]