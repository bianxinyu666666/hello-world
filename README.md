# hello-world

package nc.erm.backtask;

import java.io.BufferedReader;
import java.io.DataOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.UnsupportedEncodingException;
import java.net.HttpURLConnection;
import java.net.ProtocolException;
import java.net.URL;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.Calendar;
import java.util.Date;
import java.util.GregorianCalendar;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import com.alibaba.fastjson.JSONObject;

import nc.bs.dao.BaseDAO;
import nc.bs.framework.common.InvocationInfoProxy;
import nc.bs.logging.Logger;
import nc.bs.oa.hander.OAConfigInfo;
import nc.bs.oa.hander.OAInfo;
import nc.bs.pub.pa.PreAlertObject;
import nc.bs.pub.taskcenter.BgWorkingContext;
import nc.bs.pub.taskcenter.IBackgroundWorkPlugin;
import nc.jdbc.framework.processor.ColumnProcessor;
import nc.jdbc.framework.processor.ResultSetProcessor;
import nc.vo.pub.BusinessException;
import nc.vo.pub.lang.UFDate;
import nc.vo.pub.lang.UFDateTime;
import weaver.interfaces.encode.DES_Base64;
/**
 * 给oa发送2周内删除已办消息
 */
public class OAInfoDeleteBackPluginImpl implements IBackgroundWorkPlugin {

	@Override
	public PreAlertObject executeTask(BgWorkingContext arg0) throws BusinessException {
//		Calendar cal = Calendar.getInstance(TimeZone.getTimeZone("GMT+08:00"));
		Date now = new Date(); 
		Calendar cal = new GregorianCalendar();
		cal.setTime(now);  
		cal.setTime(new UFDate().toDate());
		cal.set(Calendar.DATE, cal.get(Calendar.DATE)-60);
		Date date = cal.getTime();
		BaseDAO dao = new BaseDAO();
		//获得已审批信息
		String sql = "select ps.code,billVersionPK,w.pk_billtype,w.messagenote from pub_workflownote w, sm_user sm,bd_psndoc ps where ischeck in('Y','X') and w.checkman=sm.cuserid and sm.pk_psndoc=ps.pk_psndoc and sm.user_type<>-1 and exists( "
					+"select 1 from ap_paybill a where billdate>='"+new UFDateTime(date).toString()+"' and a.approvestatus=1 and a.pk_paybill=w.billid "
					+"union " 
					+"select 1 from ap_payablebill b where billdate>='"+new UFDateTime(date).toString()+"' and b.approvestatus=1 and b.pk_payablebill=w.billVersionPK "
					+"union " 
					+"select 1 from ar_gatherbill c where billdate>='"+new UFDateTime(date).toString()+"' and c.approvestatus=1 and c.pk_gatherbill=w.billVersionPK "
					+"union " 
					+"select 1 from ar_recbill d where billdate>='"+new UFDateTime(date).toString()+"' and d.approvestatus=1 and d.pk_recbill=w.billVersionPK "
					+"union " 
					+"select 1 from er_bxzb e where djrq>='"+new UFDateTime(date).toString()+"' and spzt=1 and e.pk_jkbx=w.billVersionPK "  
					+"union " 
					+"select 1 from er_jkzb f where djrq>='"+new UFDateTime(date).toString()+"' and spzt=1 and f.pk_jkbx=w.billVersionPK "
					+")";
		Map<String,String> sendmap = (Map<String,String>) dao.executeQuery(sql, new ResultSetProcessor() {
			@Override
			public Object handleResultSet(ResultSet rs) throws SQLException {
				int s = 1;
				Map<String,String> map = new HashMap<String,String>();
				List<BatchSendInfo> checklist = new ArrayList<BatchSendInfo>();
				int num = 0;
				while (rs.next()) {
					num ++;
					BatchSendInfo ck = new BatchSendInfo();
					ck.setSyscode("NC");
					ck.setUserid(rs.getString(1));
					ck.setFlowid(rs.getString(2));
					checklist.add(ck);
					if(num == 300 || rs.isAfterLast()){
						map.put(s+"", JSONObject.toJSONString(checklist));
						checklist.clear();
						num = 0;
						s++;
					}
				}
				return map;
			}
			
		});
		if(sendmap != null && sendmap.size() >0){
				String surl = OAConfigInfo.getProperty("delurl");
				try {
					int num = 1;
					for(Map.Entry<String, String> entry : sendmap.entrySet()){
						sendMsg(entry.getValue(), new URL(surl));
					}
				} catch (IOException e) {
					// TODO 自动生成的 catch 块
					e.printStackTrace();
				}//发送消息
		}
		return null;
	}

	private void sendMsg(String pmas, URL url) throws IOException, ProtocolException, UnsupportedEncodingException {
		HttpURLConnection httpClient = (HttpURLConnection) url.openConnection();
		httpClient.setRequestProperty("accept", "*/*");//解决中文乱码问题
		httpClient.setDoOutput(true);
		httpClient.setDoInput(true);
		httpClient.setUseCaches(false);
		httpClient.setRequestMethod("POST");
		httpClient.setRequestProperty("connection", "Keep-Alive");
		httpClient.setRequestProperty("Content-Type", "application/json; charset=utf-8");
		DataOutputStream dos = new DataOutputStream(httpClient.getOutputStream());
		StringBuffer sb = null;
		//请求
//		String pmas = JSONObject.toJSONString(inf);
		dos.write(pmas.getBytes("utf-8"));
		dos.flush();
		dos.close();
		int resultCode = httpClient.getResponseCode();
		if (HttpURLConnection.HTTP_OK == resultCode) {
			sb = new StringBuffer();
			String readLine = new String();
			InputStream inputStream = httpClient.getInputStream();
			BufferedReader responseReader = new BufferedReader(new InputStreamReader(inputStream, "UTF-8"));
			while ((readLine = responseReader.readLine()) != null) {
				sb.append(readLine).append("\n");
			}
			responseReader.close();
			
		}else{
			
		}
	}
	
	public  OAInfo[] convertOAinf(){
		BaseDAO bs = new BaseDAO();
		OAInfo inf;
		try {
			String syscode = OAConfigInfo.getProperty("syscode");
			Object obj = null;
			if(billtype.startsWith("APCT")){
				 obj = inf.getWorkflowname();
			}else{
				String sql = "select b.billtypename from pub_wf_instance a inner join bd_billtype b on a.billtype=b.pk_billtypecode inner join org_orgs c on a.pk_org=c.pk_org inner join sm_user d on a.billmaker=d.cuserid  where billid='"+inf.getFlowid()+"'";
				 obj = dao.executeQuery(sql, new ColumnProcessor(1));
			}
			String submitter = null;
			String billcode = null;
			if(billtype.startsWith("264")){
				submitter = (String) dao.executeQuery("select sm.user_code  from er_bxzb er,sm_user sm where sm.pk_psndoc = er.jkbxr and pk_jkbx ='"+inf.getFlowid()+"'", new ColumnProcessor(1));
				billcode = (String) dao.executeQuery("select djbh from er_bxzb where pk_jkbx ='"+inf.getFlowid()+"'", new ColumnProcessor(1));
			}
			else if(billtype.startsWith("263")){
				submitter = (String) dao.executeQuery("select sm.user_code  from er_jkzb er,sm_user sm where sm.pk_psndoc = er.jkbxr and pk_jkbx ='"+inf.getFlowid()+"'", new ColumnProcessor(1));
				billcode = (String) dao.executeQuery("select djbh from er_jkzb where pk_jkbx ='"+inf.getFlowid()+"'", new ColumnProcessor(1));
			}
			else if(billtype.startsWith("262")){
				submitter = (String) dao.executeQuery("select sm.user_code  from er_accrued er,sm_user sm where sm.cuserid = er.auditman and pk_accrued_bill ='"+inf.getFlowid()+"'", new ColumnProcessor(1));		
				billcode = (String) dao.executeQuery("select billno from er_accrued where pk_accrued_bill ='"+inf.getFlowid()+"'", new ColumnProcessor(1));
			}
			else if(billtype.startsWith("261")){
				submitter = (String) dao.executeQuery("select sm.user_code from er_mtapp_bill er,sm_user sm where sm.pk_psndoc = er.billmaker and pk_mtapp_bill ='"+inf.getFlowid()+"'", new ColumnProcessor(1));
				billcode = (String) dao.executeQuery("select billno from er_mtapp_bill where pk_mtapp_bill ='"+inf.getFlowid()+"'", new ColumnProcessor(1));
			}
			else if(billtype.startsWith("F2")){
				submitter = (String) dao.executeQuery("select sm.user_code  from ar_gatherbill ar,sm_user sm where sm.cuserid = ar.billmaker and pk_gatherbill ='"+inf.getFlowid()+"'", new ColumnProcessor(1));		
				billcode = (String) dao.executeQuery("select billno from ar_gatherbill where pk_gatherbill ='"+inf.getFlowid()+"'", new ColumnProcessor(1));
			}
			else if(billtype.startsWith("APCT")){
				submitter = (String) dao.executeQuery("select sm.user_code  from sscct_paycontract ss, sm_user sm where sm.cuserid = ss.creator and pk_paycontract_v ='"+inf.getFlowid()+"'", new ColumnProcessor(1));		
				billcode = (String) dao.executeQuery("select billno from sscct_paycontract where pk_paycontract_v ='"+inf.getFlowid()+"'", new ColumnProcessor(1));
			}
			if(inf.getCreator() != null){
				inf.setCreator(submitter == null? "" : submitter.toString());
			}
			inf.setIsremark("2");//已办
			inf.setRequestname("单据处理提醒: " +(obj == null ? "" : obj.toString()) + ", 单据号: " + (billcode == null ? "" : billcode.toString())  + ", 已处理!" + "");
		
		} catch (BusinessException e) {
			Logger.error("发待办给oa数据转换：" + e);
		}
		return null;
	}

}
