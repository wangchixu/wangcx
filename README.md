
转至元数据结尾
由 王驰旭创建, 最后修改于大约1分钟以前, viewed 14 times
转至元数据起始
1、pinpoint简介
Pinpoint是一个分析大型分布式系统的平台，提供解决方案来处理海量跟踪数据。

Pinpoint的特点如下:

分布式事务跟踪，跟踪跨分布式应用的消息
自动检测应用拓扑，帮助你搞清楚应用的架构
水平扩展以便支持大规模服务器集群
提供代码级别的可见性以便轻松定位失败点和瓶颈
使用字节码增强技术，添加新功能而无需修改代码
点此了解更多：Pinpoint技术概述


pinpoint 结构图





pinpoint 主要由： 数据上报agent插件、 collector数据接收服务、webUI数据展示服务、HBase数据库 几个方面组成，如果需要用到监控，还需要一个mysq数据库

agent插件:

在需要进行监控的服务启动时候，在启动参数中添加此agent插件，该插件会定时往collector发送服务的相关数据（jvm情况、请求的数据、等）

collector【数据接收服务】：

用于接收agent插件上报过来数据，对数据进行hbase入库存储

HBase数据库：

数据进行hbase入库存储

webUI【数据展示服务】：

从hbase库中读取收集过来的各个服务的不同时刻的状态数据，进行界面化展示



具体的部署移步：PinPoint 使用 有一个部署的pdf文档

2、pinpoint监控报警配置


如何配置项目报警：

在源码下载中有个doc文件下有个一个alarm.md 文件，里面有官方报警的配置以及说明，这里替大家翻译一下

1、初始化一个mysql数据库
首先需要一个mysql数据库，用于存放报警的相关配置信息，如：哪个项目需要配置、触发报警条件，触发发送消息类型

mysql需要初始化的表结构为在源码的web项目下的resources下有个sql文件夹，有对应的脚本



2 、修改相关配置
配置文件路径主要集中在 web项目下的resources文件夹中

batch.properties
      batch.enable=true #报警开关
      batch.server.ip=127.0.0.1 #发送报警的web项目ip，防止部署多个时并发报警，一个时使用当前127.0.0.1即可


jdbc.properties
    # 第一步的数据库连接信息
    jdbc.driverClassName=com.mysql.jdbc.Driver
    jdbc.url=jdbc:mysql://localhost:13306/pinpoint?characterEncoding=UTF-8
    jdbc.username=admin
    jdbc.password=admin

applicationContext-batch-schedule.xml
<!-- 报警扫描相关配置-->
<task:scheduled-tasks scheduler="scheduler">
    <task:scheduled ref="batchJobLauncher" method="alarmJob" cron="0 0/3 * * * *" />
</task:scheduled-tasks>
<task:executor id="poolTaskExecutorForPartition" pool-size="1" />
<step id="alarmStep" xmlns="http://www.springframework.org/schema/batch">
    <tasklet task-executor="poolTaskExecutorForStep" throttle-limit="3">
        <chunk reader="reader" processor="processor" writer="writer" commit-interval="1"/>
    </tasklet>
</step>
<task:executor id="poolTaskExecutorForStep" pool-size="10" />


3、报警接口
pinpoint已经提供了报警接口在 web项目中的 AlarmMessageSender.java类中，支持短信和邮件


只要增加其实现类AlarmMessageSenderImple 即可会自动触发报警事件

这里 sendSms 和 sendEmail 只是作为触发报警的入口，报警内容需要自己自行开发，此处使用邮件报警触发入口，实际上是使用钉钉来发送报警消息



同时 pinpoint-web.properties 配置文件增加了钉钉通知的相关配置（关键字通知）

ding.talk.token=xxxxxxxxxx
ding.talk.Keyword=pp报警
pp.web.path=http://192.168.20.128:18080

 
4、报警配置操作
1、添加报警组和报警用户，填写用户相关信息（报警人，电话，邮箱等）

    



2、选择 application ，选择需要配置的应用和触发报警的规则Rule以及报警发送消息的类型Type，保存即可

    当配置的应用符合所配置的报警规则时，会触发对应的send接口，符合规则的监测事件是由上述第二点的定时任务触发的

触发报警的规则Rule解读：

SLOW COUNT / 慢请求数
当应用发出的慢请求数量超过配置阈值时触发。

SLOW RATE / 慢请求比例
当应用发出的慢请求百分比超过配置阈值时触发。

ERROR COUNT / 请求失败数
当应用发出的失败请求数量超过配置阈值时触发。

ERROR RATE / 请求失败率
当应用发出的失败请求百分比超过配置阈值时触发。

TOTAL COUNT / 总数量
当应用发出的所有请求数量超过配置阈值时触发。
   
SLOW COUNT TO CALLEE / 被调用的慢请求数量
当发送给应用的慢请求数量超过配置阈值时触发。

SLOW RATE TO CALLEE / 被调用的慢请求比例
当发送给应用的慢请求百分比超过配置阈值时触发。

ERROR COUNT TO CALLEE / 被调用的请求错误数
当发送给应用的请求失败数量超过配置阈值时触发。

ERROR RATE TO CALLEE / 被调用的请求错误率
当发送给应用的请求失败百分比超过配置阈值时触发。

TOTAL COUNT TO CALLEE / 被调用的总数量
当发送给应用的所有请求数量超过配置阈值时触发。

HEAP USAGE RATE / 堆内存使用率
当应用的堆内存使用率超过配置阈值时触发。

JVM CPU USAGE RATE / JVM CPU使用率
当应用的CPU使用率超过配置阈值时触发。




3、自定义url监控报警实现
自定义url监控报警主要是模拟调用了 web现有的接口进行接口

FilteredMapController.getFilteredServerMapDataMadeOfDotGroup 的返回结果进行数据解析
该接口的来源使用场景是 使用 Filter Wizard 进行url正则匹配查询, 从中去读取报错数量

 






实现源码如下：

package com.navercorp.pinpoint.web.alarm;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.navercorp.pinpoint.common.util.CollectionUtils;
import com.navercorp.pinpoint.common.util.DateUtils;
import com.navercorp.pinpoint.common.util.StringUtils;
import com.navercorp.pinpoint.common.util.TransactionId;
import com.navercorp.pinpoint.web.alarm.checker.AlarmChecker;
import com.navercorp.pinpoint.web.applicationmap.ApplicationMap;
import com.navercorp.pinpoint.web.applicationmap.FilterMapWrap;
import com.navercorp.pinpoint.web.applicationmap.nodes.Node;
import com.navercorp.pinpoint.web.filter.Filter;
import com.navercorp.pinpoint.web.filter.FilterBuilder;
import com.navercorp.pinpoint.web.filter.FilterDescriptor;
import com.navercorp.pinpoint.web.service.FilteredMapService;
import com.navercorp.pinpoint.web.service.UserGroupService;
import com.navercorp.pinpoint.web.util.LimitUtils;
import com.navercorp.pinpoint.web.vo.LimitedScanResult;
import com.navercorp.pinpoint.web.vo.Range;
import org.apache.hadoop.hbase.shaded.org.apache.http.HttpResponse;
import org.apache.hadoop.hbase.shaded.org.apache.http.client.HttpClient;
import org.apache.hadoop.hbase.shaded.org.apache.http.client.methods.HttpPost;
import org.apache.hadoop.hbase.shaded.org.apache.http.entity.StringEntity;
import org.apache.hadoop.hbase.shaded.org.apache.http.impl.client.DefaultHttpClient;
import org.apache.hadoop.hbase.shaded.org.apache.http.util.EntityUtils;
import org.apache.hadoop.hbase.shaded.org.mortbay.util.ajax.JSON;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.ClassPathResource;
import org.springframework.util.Base64Utils;
import org.springframework.web.bind.annotation.RequestParam;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;
import java.io.IOException;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.*;

/**
 * Created by wangcx on 2019年12月31日 19:15:48
 */
public class AlarmMessageSenderImple implements AlarmMessageSender {


    //钉钉机器人token
    @Value("#{pinpointWebProps['ding.talk.token']}")
    private String dingTalkToken;
    //钉钉机器人关键字
    @Value("#{pinpointWebProps['ding.talk.Keyword']}")
    private String dingTalkKeyword;
    //pp地址:到端口号即可
    @Value("#{pinpointWebProps['pp.web.path']}")
    private String ppWebPath;

    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    @Autowired
    UserGroupService userGroupService;

    @Autowired
    private FilterBuilder filterBuilder;

    @Autowired
    private FilteredMapService filteredMapService;

    //6ec22dcf568a0707a3498293855929fcc46c6fe14880e4ecb2b34d4c5a6e36a6
    private  String WEBHOOK_TOKEN = "https://oapi.dingtalk.com/robot/send?access_token=";

    @Override
    public void sendSms(AlarmChecker checker, int sequenceCount) {

        List<String> receivers = userGroupService.selectPhoneNumberOfMember(checker.getuserGroupId());
        logger.info(" =============准备发送消息=============== " + receivers.size());

        if (receivers.size() == 0) {
            return;
        }
        List<String> smsList = checker.getSmsMessage();
        for (String message : smsList) {
            logger.info("send SMS : {}", message);
            // TODO Implement logic for sending SMS
            for (String receiver : receivers) {
                logger.info("send email receiver : {}", receiver);
            }
        }
    }

    @Override
    public void sendEmail(AlarmChecker checker, int sequenceCount) {

        List<String> receivers = userGroupService.selectEmailOfMember(checker.getuserGroupId());
        logger.info(" ==============准备发送邮件=============== " + receivers.size());

        if (receivers.size() == 0) {
            return;
        }
        String message = checker.getEmailMessage();
        logger.info("checker : {}", JSON.toString(checker.getRule()));
        logger.info("send email : {}", message);

        // TODO Implement logic for sending email
        for (String receiver : receivers) {
            logger.info("send email receiver : {}", receiver);
        }
        try {
            String errorMsessage = customAlarmCheck(checker.getRule().getApplicationId());
            logger.info("自定义报警调用返回 customAlarmCheck={} " ,errorMsessage);

            if(!StringUtils.isEmpty(errorMsessage)){
                sendDingTalk(dingTalkKeyword +":\n" + errorMsessage);
            }
        }catch (Exception e){
            logger.info("sendDingTalk Exception : {}", e.getMessage());
        }

    }

    private String customAlarmCheck(String applicationId) throws IOException {

        LocalDateTime now = LocalDateTime.now();
        String jsonMapText = readFromCustomProperties();
        ObjectMapper mapper = new ObjectMapper();
        Map<String,List<String>> jsonMapObj  = mapper.readValue(jsonMapText, Map.class);
        //Map<String,List<String>> jsonMapObj = (Map<String,List<String>>)JSON.parse(jsonMapText);
        List<String> urlList = jsonMapObj.get(applicationId);
        StringBuffer errorMsg =  new StringBuffer();
        for (String url : urlList) {
            String requestUrl = "";
            String urlMsg = "";
            String[]  urlMsgArray = url.split("#");
            if(null != urlMsg && urlMsgArray.length > 0 ){
                for (int i=0; i < urlMsgArray.length; i++) {
                    if(i==0){
                        requestUrl = urlMsgArray[0];
                    }else if(i==1){
                        urlMsg = urlMsgArray[1];
                    }

                }
            }
            logger.info("报警调用返回错误信息内容 getErrorMsgFromUserByApplicationName applicationId ={},requestUrl ={}", applicationId,requestUrl);
            String errMsg = getErrorMsgFromUserByApplicationName(applicationId,requestUrl);
            logger.info("报警调用返回错误信息内容 getErrorMsgFromUserByApplicationName errMsg : {}", errMsg);

            if(!StringUtils.isEmpty(errMsg)){
                if(!StringUtils.isEmpty(urlMsg)){
                    errorMsg.append(urlMsg).append("接口报错-->");
                }
                errorMsg.append(errMsg);
            }

        }
   /*
   /#/main/t-uama-api-management@SPRING_BOOT/100m/2020-01-10-23-39-56
    */
        if(!StringUtils.isEmpty(errorMsg.toString())){
            DateTimeFormatter formatter2 = DateTimeFormatter.ofPattern("yyyy-MM-dd-HH-mm-ss");
            errorMsg.append(ppWebPath + "/#/main/"+applicationId+"@SPRING_BOOT/5m/"+ now.format(formatter2));
        }
        return errorMsg.toString();
    }

    private  String readFromCustomProperties() {
        String alarmJson="";
        try {

            // ClassPathResource类的构造方法接收路径名称，自动去classpath路径下找文件
            ClassPathResource classPathResource = new ClassPathResource("custom/alarmJson.json");

            // 获得File对象，当然也可以获取输入流对象
            File file = classPathResource.getFile();

            BufferedReader bufferedReader = new BufferedReader(new FileReader(file));
            StringBuilder content = new StringBuilder();
            String line = null;
            while ((line = bufferedReader.readLine()) != null) {
                content.append(line);
            }
            alarmJson = content.toString();
            bufferedReader.close();
        } catch (IOException e) {
            e.printStackTrace();
            logger.info("读取自定义配置错误，使用默认配置-->{\"api-management\": [\"/manage/**\"]}");
            alarmJson= "{\"api-management\":[\"/manage/**\"]}";
        }
        return alarmJson;
    }




    private  void sendDingTalk(String content) throws Exception{

        HttpClient httpclient = new DefaultHttpClient();

        HttpPost httppost = new HttpPost(WEBHOOK_TOKEN + dingTalkToken);
        httppost.addHeader("Content-Type", "application/json; charset=utf-8");
        String textMsg = "{ \"msgtype\": \"text\", \"text\": {\"content\": \"" + content + "\"}}";
        StringEntity se = new StringEntity(textMsg, "utf-8");
        httppost.setEntity(se);
        logger.info("ding发送报警内容===========url:{}======textMsg:{},",WEBHOOK_TOKEN + dingTalkToken,textMsg);
        HttpResponse response = httpclient.execute(httppost);
        String result= EntityUtils.toString(response.getEntity(), "utf-8");
        logger.info("发送报警完毕================={}",result);

    }


    /**
     * 查询入口服务的报错信息
     * @param applicationId
     * @return
     */
    private String getErrorMsgFromUserByApplicationName(String applicationId,String url) {
        try {

            Calendar endTime = Calendar.getInstance();
            Calendar beginTime = Calendar.getInstance();
            beginTime.add(Calendar.MINUTE, -5);// 5分钟之前的时间
            Long from = beginTime.getTimeInMillis();
            Long to = endTime.getTimeInMillis();

            FilterDescriptor filterDescriptor = new FilterDescriptor();
            //[{"fa":"USER","fst":"USER","ta":"api-management","tst":"SPRING_BOOT","ie":null,"url":"L21hbmFnZS92aXNpdG9yL2xpc3Q="}]
            filterDescriptor.setFromApplicationName("USER");
            filterDescriptor.setFromServiceType("USER");
            filterDescriptor.setToApplicationName(applicationId);
            filterDescriptor.setToServiceType("SPRING_BOOT");
            filterDescriptor.setUrl(new String(Base64Utils.encode(url.getBytes(StandardCharsets.UTF_8.name())),StandardCharsets.UTF_8.name()));
            logger.info("自定义监测报警对象url" + JSON.toString(filterDescriptor));
            List<FilterDescriptor> obj = new ArrayList<FilterDescriptor>();
            obj.add(filterDescriptor);
            ObjectMapper mapper = new ObjectMapper();
            String filterJson = mapper.writeValueAsString(obj);

            String filterText = URLEncoder.encode(filterJson,StandardCharsets.UTF_8.name());
            obj.add(filterDescriptor);
            FilterMapWrap  resultMapWrap = getFilteredServerMapDataMadeOfDotGroup(applicationId,"",from,to,to,1000,1,filterText,null,10000,0);
            Long errCount = parsingErrorMsgFromFilterMapWrapData(resultMapWrap,applicationId);
            logger.info("报警调用返回结果 parsingErrorMsgFromFilterMapWrapData errCount : {}", errCount);
            return (errCount>0) ?
                    ( applicationId +"-->"+ url +"-->"+ "errCount:" + errCount +";\\n")
                    : "";

        } catch (Exception e) {
            logger.info("报警调用查询 getFilteredServerMapDataMadeOfDotGroup error : {}", e.getMessage());
            e.printStackTrace();
        }

        return "报警失败!";
    }

    private Long parsingErrorMsgFromFilterMapWrapData(FilterMapWrap resultMapWrap,String applicationName) {
        Long errCount = 0L;
        List<Node> nodeList = new ArrayList(resultMapWrap.getApplicationMap().getNodes());
        logger.info("nodeList={}",JSON.toString(nodeList));
        if(!CollectionUtils.isEmpty(nodeList)){

           /* for (Node node : nodeList) {
                logger.info("node={}",JSON.toString(node));
                logger.info("getApplicationTextName={}",node.getApplicationTextName());
                logger.info("TotalErrorCount={}",node.getNodeHistogram().getApplicationHistogram().getTotalErrorCount());
                if(applicationName.equals(node.getApplicationTextName())){
                    if(node.getNodeHistogram().getApplicationHistogram().getTotalErrorCount() > 0){
                        errCount = errCount +  node.getNodeHistogram().getApplicationHistogram().getTotalErrorCount();
                    }
                }
            }*/

            Optional<Node> node = nodeList.stream().filter(item -> applicationName.equals(item.getApplicationTextName())).findFirst();
            if(node.isPresent()){
                logger.info("node.getApplicationTextName={}",node.get().getApplicationTextName());
                logger.info("node.TotalErrorCount={}",node.get().getNodeHistogram().getApplicationHistogram().getTotalErrorCount());
                if(node.get().getNodeHistogram().getApplicationHistogram().getTotalErrorCount() > 0){
                    errCount = errCount +  node.get().getNodeHistogram().getApplicationHistogram().getTotalErrorCount();
                }
            }
        }

        return errCount;
    }


    /**
     * copy from FilteredMapController.getFilteredServerMapDataMadeOfDotGroup
     * @param applicationName
     * @param serviceTypeName
     * @param from
     * @param to
     * @param originTo
     * @param xGroupUnit
     * @param yGroupUnit
     * @param filterText
     * @param filterHint
     * @param limit
     * @param viewVersion
     * @return
     * @throws Exception
     */
    private FilterMapWrap getFilteredServerMapDataMadeOfDotGroup(@RequestParam("applicationName") String applicationName,
                             @RequestParam("serviceTypeName") String serviceTypeName,
                             @RequestParam("from") long from,
                             @RequestParam("to") long to,
                             @RequestParam("originTo") long originTo,
                             @RequestParam("xGroupUnit") int xGroupUnit,
                             @RequestParam("yGroupUnit") int yGroupUnit,
                             @RequestParam(value = "filter", required = false) String filterText,
                             @RequestParam(value = "hint", required = false) String filterHint,
                             @RequestParam(value = "limit", required = false, defaultValue = "10000") int limit,
                             @RequestParam(value = "v", required = false, defaultValue = "0") int viewVersion) throws Exception{
        logger.info("applicationName={},serviceTypeName={},from={},to={},originTo={},xGroupUnit={},yGroupUnit={},filter={},hint={},limit={},v={}",
                applicationName,serviceTypeName,from,to,originTo,xGroupUnit,yGroupUnit,filterText,filterHint,limit,viewVersion);

        if (xGroupUnit <= 0) {
            throw new IllegalArgumentException("xGroupUnit(" + xGroupUnit + ") must be positive number");
        }
        if (yGroupUnit <= 0) {
            throw new IllegalArgumentException("yGroupUnit(" + yGroupUnit + ") must be positive number");
        }
 

为了对需要监控的接口进行配置，增加了一个位置文件，resource/custom/alarmJson.json

用于配置，哪个项目，哪些接口需要监控

json数据格式样例(【#】分割，设置为接口说明)：

{
   "t-uama-api-lc-community":
   [
      "/community/notice/getPropertyNoticeNum#获取用户项目通知",
      "/community/notice/getUserNoticeNum#获取用户未读通知数",
      "/community/home/getUserInfoAndAsset#用户信息及资产",
      "/community/home/getYzAppHome#app首页",
      "/community/businessMould/businessValue/list#获取业务域值以及换肤",
      "/community/login/getAppPatch#获取版本补丁",
      "/community/login/saveLoginLog#记录登录日志",
      "/community/login/getAppVersion#获取APP版本信息",
      "/community/order/insertOrderInfo#提交表单服务订单",
      "/community/visitor/passCode#生成访客通行证",
      "/community/visitor/info#再次邀约接口",
      "/community/order/insertNewOrderInShoppingCart#商品订单下单、结算购物车",
      "/community/pay/getAlipayInfo/product#支付宝App支付"
   ],
   "t-uama-api-lc-h5":
   [
      "/h5api/operationPosition#运营位列表"
   ],
    "t-uama-api-management":
    [
      "/manage/product/region/list/product/app#获得产品域APP信息列表",
      "/manage/businessMouldInfo/app/download/page/info#获取下载页面信息"
    ]
}
