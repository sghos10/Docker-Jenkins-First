public String extractMessageAsString(Message message) throws JMSException {
    if (message instanceof TextMessage) {
        return ((TextMessage) message).getText();
    } else if (message instanceof BytesMessage) {
        BytesMessage bytesMessage = (BytesMessage) message;
        byte[] data = new byte[(int) bytesMessage.getBodyLength()];
        bytesMessage.readBytes(data);
        return new String(data);
    } else if (message instanceof ObjectMessage) {
        ObjectMessage objMsg = (ObjectMessage) message;
        Object obj = objMsg.getObject();
        return obj != null ? obj.toString() : null;
    } else {
        return null; // or throw exception
    }
}
