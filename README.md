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

