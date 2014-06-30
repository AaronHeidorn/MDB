MDB
===

This is the producer that sets messages on a queue for an MDB.  Exception handling, comments, and loggers have been removed




	Context ctx = new InitialContext();
	JMSConnectionFactory = (ConnectionFactory) ctx.lookup("jms/MQFactory");
			  
	conn = JMSConnectionFactory.createConnection();
	session = conn.createSession(false, Session.AUTO_ACKNOWLEDGE);
	Destination destination = session.createQueue(qName);
	MessageProducer producer = session.createProducer(destination);

	producer.send(session.createObjectMessage(message));


This is the entire MDB class:

	package com.bcbst.batchworker.jms;
	
	import javax.ejb.ActivationConfigProperty;
	import javax.ejb.MessageDriven;
	import javax.ejb.TransactionAttribute;
	import javax.ejb.TransactionAttributeType;
	import javax.ejb.TransactionManagement;
	import javax.ejb.TransactionManagementType;
	import javax.jms.JMSException;
	import javax.jms.Message;
	import javax.jms.MessageListener;
	import javax.jms.ObjectMessage;
	
	import org.apache.log4j.Logger;
	
	import com.bcbst.batchapplicationutilities.utils.BatchMessage;
	import com.bcbst.batchapplicationutilities.utils.Utils;
	
	/**
	 * Message-Driven Bean implementation class for: MessageDrivenBean
	 *
	 */
	@MessageDriven(
			  name = "MessageDrivenBean",
			  activationConfig = {
			    @ActivationConfigProperty(propertyName  = "destinationType", 
			                              propertyValue = "javax.jms.Queue"),
			    @ActivationConfigProperty(propertyName  = "destination", 
			                              propertyValue = "jms/EBSDBatch")
			  }
			)
	@TransactionManagement(TransactionManagementType.CONTAINER)
	@TransactionAttribute(TransactionAttributeType.REQUIRED)
	public class MessageDrivenBean implements MessageListener {
		private Logger logger = Utils.getLogger(MessageDrivenBean.class);
	
	    
	    
		/**
	     * @see MessageListener#onMessage(Message)
	     */
	    public void onMessage(Message message) {
	    	
		    try {
		    	if (message instanceof ObjectMessage) {
			       
			            ObjectMessage jmsMessage = (ObjectMessage) message;
			            
						logger.trace("onMessage -> extracting BatchMessage from ObjectMessage");
						BatchMessage batchMessage = (BatchMessage) jmsMessage.getObject();
						
						logger.trace("onMessage -> got BatchMessage from ObjectMessage. executing run().");
						batchMessage.run();
		    	} else {
		    		logger.error("onMessage -> The recieved message was not an ObjectMessage");
		    	}
			} catch (JMSException e) {
				logger.error("onMessage -> Error extracting or acknowledging message", e);
			} 
	    }
	}


This is the Message.Run() method currently being executed

	public static void run(String userId) {
		long time = 0;
		int repetitionNumber = Integer.valueOf(userId.substring(12));
		
		time = executeInsert(userId);
		
		if (time >= 200) {
			logger.debug("repetition: " + repetitionNumber + " time: " + time + "------------ABNORMAL------------");
		} else if (time < 0)  {
			logger.debug("repetition: " + repetitionNumber + " ERROR");
		} else if (repetitionNumber % 1000 == 0) {
			logger.debug("repetition: " + repetitionNumber + " time: " + time);
		} 
	}
	
	private static long executeInsert(String userId) {
		long starttime = 0;
		long endtime = 0;
		long totaltime = -1;
		
		Connection conn = null;
		PreparedStatement ps = null;
		String memberInsertSql =  	"INSERT INTO " +
				"MEMBER_LOADTEST(" +
				"WEBUSERID, SUBSCRIBERID, FIRSTNAME, MIDDLEINITIAL," +
				"LASTNAME, ADDRESSLINE1, ADDRESSLINE2, CITY, " +
				"STATE, ZIP, REGISTERDATE, LETTERGENERATEDFLAG, " +
				"LETTERGENERATIONDATE, GROUPNO, OTHERMEMBERSPECIFICDATA,eMailAddress ) " +
				"VALUES(" + 
				"?, ?, ?, ?, " + 
				"?, ?, ?, ?, " +
				"?, ?, ?, ?, " +
				"?, ?, ? , ? )"  ;
		
		try {
			conn = Conn.getEBSDPracticeConnection();
			ps = conn.prepareStatement(memberInsertSql);
			
			ps.setString(1, userId);
			ps.setString(2, "12345678");
			ps.setString(3, "first");
			ps.setString(4, "m");
			ps.setString(5, "last");
			ps.setString(6, "addr1");
			ps.setString(7, "addr2");
			ps.setString(8, "city");
			ps.setString(9, "ST");
			ps.setString(10, "12345");
			ps.setTimestamp(11, new Timestamp(starttime));
			ps.setString(12, "N");
			ps.setString(13, null);
			ps.setString(14, "12345");
			ps.setString(15, null);
			ps.setString(16, null);

			starttime = System.currentTimeMillis();
			ps.executeUpdate();
			endtime = System.currentTimeMillis();
			
			totaltime = endtime - starttime;
			
		} catch (SQLException e) {
			logger.error("-----------ERROR WITH ADD---------", e);
		} catch (NamingException e) {
			logger.error("-----------UNABLE TO FIND DATASOURCE---------", e);
		} finally {
			Conn.cleanup(conn);
			Conn.cleanup(ps);
		}
		
		return totaltime;
	}
