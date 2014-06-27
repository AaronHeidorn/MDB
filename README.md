MDB
===

This is the producer that sets messages on a queue for an MDB.  Exception handling, comments, and loggers have been removed




	Context ctx = new InitialContext();
	JMSConnectionFactory = (ConnectionFactory) ctx.lookup("jms/MQFactory");
			  
	conn = JMSConnectionFactory.createConnection();
	session = conn.createSession(false, Session.CLIENT_ACKNOWLEDGE);
	Destination destination = session.createQueue(qName);
	MessageProducer producer = session.createProducer(destination);

	producer.send(session.createObjectMessage(message));


This is the entire MDB class:

	package com.bcbst.batchworker.jms;
	
	import javax.annotation.Resource;
	import javax.ejb.ActivationConfigProperty;
	import javax.ejb.MessageDriven;
	import javax.ejb.MessageDrivenContext;
	import javax.ejb.TransactionManagement;
	import javax.ejb.TransactionManagementType;
	import javax.jms.JMSException;
	import javax.jms.Message;
	import javax.jms.MessageListener;
	import javax.jms.ObjectMessage;
	import javax.transaction.HeuristicMixedException;
	import javax.transaction.HeuristicRollbackException;
	import javax.transaction.RollbackException;
	import javax.transaction.SystemException;
	
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
	@TransactionManagement(TransactionManagementType.BEAN)
	public class MessageDrivenBean implements MessageListener {
		private Logger logger = Utils.getLogger(MessageDrivenBean.class);
	
	    @Resource
	    private MessageDrivenContext mdc;
	    
		
		public void ejbCreate() {
			try{
				logger.warn("Committing global transaction on MDB upon creation");
				
				//remove global transaction from MDB
				//transactions are needed on a message by message basis - 
				//not globally on the MDB itself

				//This is a 'hack' to get around global transactions timing out
				//This is what I'm trying to avoid/fix
				mdc.getUserTransaction().commit();
				
			} catch (SecurityException e) {
	    		logger.error("onMessage -> Error committing transaction", e);
			} catch (IllegalStateException e) {
	    		logger.error("onMessage -> Error committing transaction", e);
			} catch (RollbackException e) {
	    		logger.error("onMessage -> Error committing transaction", e);
			} catch (HeuristicMixedException e) {
	    		logger.error("onMessage -> Error committing transaction", e);
			} catch (HeuristicRollbackException e) {
	    		logger.error("onMessage -> Error committing transaction", e);
			} catch (SystemException e) {
	    		logger.error("onMessage -> Error committing transaction", e);
			}
		}
	    
	    
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
		    	
		    	//always acknowledge message once to this point.  If processing fails in the run() 
		    	//method, the appropriate message should be recreated and placed on an error queue
		    	message.acknowledge();
				
			} catch (JMSException e) {
				logger.error("onMessage -> Error extracting or acknowledging message", e);
			} 
	    }
	}
