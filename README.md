package com.truist.payflow.listener;

import java.time.LocalDateTime;
import java.util.Objects;
import java.util.UUID;
import java.util.logging.Logger;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.dao.TransientDataAccessException;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.jms.core.MessageCreator;
import org.springframework.jms.listener.SessionAwareMessageListener;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Component;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.JsonMappingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.truist.payflow.dto.ZelleMQPaymentResponse;
import com.truist.payflow.util.RtpStatusMap;
import com.truist.pf.enums.State;
import com.truist.pf.enums.Status;
import com.truist.pf.event.entity.TblAuditEventStore;
import com.truist.pf.event.repository.EventHubStoreRepository;
import com.truist.pf.event.service.PayflowEventService;
import com.truist.pf.model.EventStore;
import com.truist.pf.model.GatewayResponseData;

import com.truist.pf.model.PayflowStatus;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Recover;

import jakarta.jms.BytesMessage;
import jakarta.jms.Destination;
import jakarta.jms.JMSException;
import jakarta.jms.Message;
import jakarta.jms.ObjectMessage;
import jakarta.jms.Session;
import jakarta.jms.TextMessage;

@Component
//@RequiredArgsConstructor
public class PayflowZelleMqListenerService implements SessionAwareMessageListener {

	private static final Logger LOGGER = Logger.getLogger(PayflowZelleMqListenerService.class.getName());

	private static final ObjectMapper objectMapper = new ObjectMapper().registerModule(new JavaTimeModule())
			.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false)
			.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

	// @Autowired(required = true)
	private final PayflowEventService payflowEventService;

	private EventHubStoreRepository eventStoreRepository;

	@Value("${tov.mq.tovSenderQueueName}")
	private String tovSenderQueueName;

	@Value("${cps.mq.zelleDlqName}")
	private String zelleDlqName;

	@Value("${cps.mq.dlqEnabled}")
	private boolean DLQEnabled;

	/*
	 * @Autowired
	 * 
	 * @Qualifier("tovSenderJmsTemplate")
	 */
	JmsTemplate tovSenderJmsTemplate;

	// @Autowired
	RtpStatusMap rtpStatusMap;

	public PayflowZelleMqListenerService(PayflowEventService payflowEventService,
			EventHubStoreRepository eventStoreRepository, JmsTemplate tovSenderJmsTemplate, RtpStatusMap rtpStatusMap) {
		super();
		this.payflowEventService = payflowEventService;
		this.eventStoreRepository = eventStoreRepository;
		this.tovSenderJmsTemplate = tovSenderJmsTemplate;
		this.rtpStatusMap = rtpStatusMap;
	}

	@Override
	@Retryable(value = { JMSException.class,
			TransientDataAccessException.class }, maxAttempts = 1, backoff = @Backoff(delay = 1000)

	)
	public void onMessage(Message message, Session session) throws JMSException {
		LOGGER.info("response message: " + message);
		TextMessage txt = null;
		ZelleMQPaymentResponse zellePaymentResponse = null;
		EventStore convertToEventStore = null;
		GatewayResponseData updateEventFromGateWay = null;

		try {
			txt = (TextMessage) message;
			LOGGER.info(txt.getText());
			zellePaymentResponse = objectMapper.readValue(txt.getText(), ZelleMQPaymentResponse.class);

			LOGGER.info("ResponseQueue Entry in Gateway ");

			convertToEventStore = ConvertToEventStore(zellePaymentResponse);

			try {
				updateEventFromGateWay = payflowEventService.updateEventFromGateWay(convertToEventStore);
			} catch (Exception e) {

				throw new Exception();

			}
			String writeValueAsString = objectMapper.writeValueAsString(zellePaymentResponse);

			pushToTOVQueue(writeValueAsString, convertToEventStore.getUETR());

			LOGGER.info(" --exit Gateway -- ");
		} catch (JsonProcessingException e) {

			putMessageToDLQ(message);

		} catch (TransientDataAccessException e) {

			throw e;

		} catch (JMSException e) {

			throw new JMSException("JMS Exception Occured..!");
		} catch (Exception e) {

			putMessageToDLQ(message);

		}

	} 

	/**
	 * To push meesage in TOV queue
	 * 
	 * @param tovMessage
	 * @param correlationID
	 */
	private void pushToTOVQueue(String tovMessage, String correlationID) {
		tovSenderJmsTemplate.send(tovSenderQueueName, new MessageCreator() {

			@Override
			public Message createMessage(Session session) throws JMSException {
				LOGGER.info("Start : publishing to Tov MQ ");
				Message message = session.createTextMessage(tovMessage);
				message.setJMSCorrelationID(correlationID);
				message.setStringProperty("Content_Type", "application/json");
				message.setStringProperty("mediaType", "application/json");

				TextMessage muleRequest = (TextMessage) message;
				LOGGER.info("End : Published To TOV MQ : " + tovSenderQueueName + message.toString());
				return message;

			}
		});
	}

	@Recover
	public void recover(Exception e, Message message, Session session) throws JMSException {

		putMessageToDLQ(message);
		LOGGER.info("End : Published To DLQ");

	}

	private void putMessageToDLQ(Message message) throws JMSException {

		if(DLQEnabled) { 
		
		
		String MessageString = extractMessageAsString(message);
		String jmsCorrelationID = message.getJMSCorrelationID();

		tovSenderJmsTemplate.send(zelleDlqName, new MessageCreator() {

			@Override
			public Message createMessage(Session session) throws JMSException {
				Message MQmessage = session.createTextMessage(MessageString);
				MQmessage.setJMSCorrelationID(jmsCorrelationID);
				MQmessage.setStringProperty("Content_Type", "application/json");
				MQmessage.setStringProperty("mediaType", "application/json");
				MQmessage.setStringProperty("DLQ_REASON", "Due to exception messages are put to DLQ..!");
				LOGGER.info("----------Message Published to DLQ --------------------");
				return MQmessage;
			}

		});
		
		}
		else {
			
			LOGGER.info("----------Message not Published to DLQ --------------------");
			
		}
	}

	private EventStore ConvertToEventStore(ZelleMQPaymentResponse response) {
		// TODO Auto-generated method stub

		EventStore eventStore = new EventStore();

		eventStore.setUpdatedTimestamp(LocalDateTime.now());

		if (Objects.nonNull(response)) {
			if (Objects.nonNull(response.getMetaData())) {
				if (Objects.nonNull(response.getMetaData().getUETR())) {
					eventStore.setTransactionId(response.getMetaData().getUETR());
					eventStore.setUETR(response.getMetaData().getUETR());
				}
				if (Objects.nonNull(response.getMetaData().getPaymentStatus())) {
					String paymentStatus = paymentStatus(response.getMetaData().getPaymentStatus());
					if (Objects.nonNull(paymentStatus)) {
						Status statusEnum = Status.fromString(paymentStatus);
						if (Objects.nonNull(statusEnum)) {
							eventStore.setStatus(statusEnum);
						}
					}
				}
			}
			if (Objects.nonNull(response.getErrorCode())) {
				eventStore.setReasonCd(response.getErrorCode());
			}
			if (Objects.nonNull(response.getErrorDescription())) {
				eventStore.setReasonDesc(response.getErrorDescription());
			}
		}
//		eventStore.setStatus(
//				response.getMetaData().getPaymentStatus().equals("Sent")
//						? Status.PAYMENT_COMPLETED :
//								response.getMetaData().getPaymentStatus().equals("Delivered") 
//								? Status.PAYMENT_COMPLETED :
//										response.getMetaData().getPaymentStatus().equals("Failed")
//										? Status.PAYMENT_REJECTED :
//												response.getMetaData().getPaymentStatus().equals("Denied")
//												? Status.PAYMENT_REJECTED :
//														response.getMetaData().getPaymentStatus().equals("Hold")
//														? Status.IN_PROGRESS :
//																response.getMetaData().getPaymentStatus().equals("PENDING")
//																? Status.IN_PROGRESS
//																		: null);
		return eventStore;

	}

	public String paymentStatus(String paymentStatus) {
		String upperCaseStatus = "";
		if (Objects.nonNull(paymentStatus)) {
			upperCaseStatus = paymentStatus.trim().toUpperCase();
		}
		return upperCaseStatus;
	}

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
			return null; // or throw exception } }
		}
	}

}
