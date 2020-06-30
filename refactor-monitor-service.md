#### 背景
  由于现在的monitor service是配置在任务维度的，且端口需要不同。当任务数增加后，端口数也随之增加。Monitor service最好应该独立配置，相同的注册中心下，配置一个即可。当任务出现调度异常需要dump信息时，使用'dump@jobName'格式进行任意任务的注册信息输出。

#### 设计和实现
  1. 支持spring的xml文件配置，新增monitor配置标签，如下：

     ```
     //registry-center-ref为必须，port可选(-1即不启动)。
     <monitor:embed id="monitor1" registry-center-ref="regCenter" monitor-port="-1"/>
     ``` 
  2. 新增monitor.xsd
  
     ```
     <xsd:schema xmlns="http://elasticjob.shardingsphere.apache.org/schema/monitor"
        xmlns:xsd="http://www.w3.org/2001/XMLSchema"
        xmlns:beans="http://www.springframework.org/schema/beans"
        targetNamespace="http://elasticjob.shardingsphere.apache.org/schema/monitor"
        elementFormDefault="qualified"
        attributeFormDefault="unqualified">
     <xsd:import namespace="http://www.springframework.org/schema/beans"/>
    
     <xsd:element name="embed">
        <xsd:complexType>
            <xsd:complexContent>
                <xsd:extension base="beans:identifiedType">
                    <xsd:attribute name="registry-center-ref" type="xsd:string" use="required" />
                    <xsd:attribute name="monitor-port" type="xsd:string" default="-1"/>
                </xsd:extension>
            </xsd:complexContent>
        </xsd:complexType>
      </xsd:element>
     </xsd:schema>
     ```
  3. 修改spring.handlers和spring.schemas

     ```
      http://elasticjob.shardingsphere.apache.org/schema/monitor=org.apache.shardingsphere.elasticjob.lite.spring.monitor.handler.MonitorNamespaceHandler
     ```

  4. 修改spring.schemas
     ```
     http\://elasticjob.shardingsphere.apache.org/schema/monitor/monitor.xsd=META-INF/namespace/monitor.xsd
     ```
  5. 新增解析类MonitorNamespaceHandler，EmbedMonitorBeanDefinitionParser，MonitorBeanDefinitionParserTag
     ```
     public final class MonitorNamespaceHandler extends NamespaceHandlerSupport {
    
       @Override
       public void init() {
          registerBeanDefinitionParser("embed", new EmbedMonitorBeanDefinitionParser());
       }
     }

     public class EmbedMonitorBeanDefinitionParser extends AbstractBeanDefinitionParser {

        @Override
        protected AbstractBeanDefinition parseInternal(final Element element, final ParserContext parserContext) {
            BeanDefinitionBuilder result = BeanDefinitionBuilder.rootBeanDefinition(MonitorService.class);
        result.addConstructorArgReference(element.getAttribute(MonitorBeanDefinitionParserTag.REGISTRY_CENTER_REF_ATTRIBUTE));
        result.addConstructorArgValue(element.getAttribute(MonitorBeanDefinitionParserTag.MONITOR_PORT_ATTRIBUTE));
        result.setInitMethodName("listen");
        result.setDestroyMethodName("close");
        return result.getBeanDefinition();
       }
     }
     
     public final class MonitorBeanDefinitionParserTag {
    
        public static final String REGISTRY_CENTER_REF_ATTRIBUTE = "registry-center-ref";
    
        public static final String MONITOR_PORT_ATTRIBUTE = "monitor-port";
     }
     ```
   6. MonitorService逻积不变。稍作调整
      ```
      public final class MonitorService {
    
        public static final String DUMP_COMMAND = "dump";
    
        private final int port;
    
        private final CoordinatorRegistryCenter regCenter;
    
        private ServerSocket serverSocket;
    
        private volatile boolean closed;
    
        public MonitorService(final CoordinatorRegistryCenter regCenter, final int port) {
          this.port = port;
          this.regCenter = regCenter;
        }
    
        /**
         * start to listen.
        */
        public void listen() {
          if (port < 0) {
              return;
          }
          try {
              log.info("Elastic job: Monitor service is running, the port is '{}'", port);
              openSocketForMonitor(port);
          } catch (final IOException ex) {
              log.error("Elastic job: Monitor service listen failure, error is: ", ex);
          }
        }
      ```
   7. 去掉job的monitor port配置，包括xsd和java代码。
   8. 修改testcase。