MDB
===

This is sample code from a project involving MDBs




	Context ctx = new InitialContext();
	JMSConnectionFactory = (ConnectionFactory) ctx.lookup("jms/MQFactory");
			  
	conn = JMSConnectionFactory.createConnection();
session = conn.createSession(false, Session.CLIENT_ACKNOWLEDGE);
	Destination destination = session.createQueue(qName);
	MessageProducer producer = session.createProducer(destination);
