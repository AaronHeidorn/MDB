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
