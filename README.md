import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.mockito.junit.MockitoJUnitRunner;

import javax.jms.*;
import static org.mockito.Mockito.*;

@RunWith(MockitoJUnitRunner.class)
public class PayflowZelleMqListenerServiceTest {

    @InjectMocks
    private PayflowZelleMqListenerService listenerService;

    @Mock
    private PayflowEventService payflowEventService;

    @Mock
    private JmsTemplate jmsTemplate;

    @Mock
    private Session session;

    @Mock
    private TextMessage textMessage;

    @Before
    public void setUp() throws Exception {
        MockitoAnnotations.initMocks(this);
        // enable DLQ for most tests
        org.springframework.test.util.ReflectionTestUtils.setField(listenerService, "DLQEnabled", true);
    }

    @Test
    public void testHappyPath_Sent() throws Exception {
        when(textMessage.getText()).thenReturn("payload");

        EventStore store = new EventStore();
        store.setUETR("123");
        store.setPaymentStatus("Sent");

        GatewayResponseData resp = new GatewayResponseData();
        resp.setEventStore(store);

        when(payflowEventService.updateEventFromGateWay(anyString())).thenReturn(resp);

        listenerService.onMessage(textMessage, session);

        verify(jmsTemplate).send(eq("TOV_QUEUE"), any());
    }

    @Test
    public void testNullEventStore() throws Exception {
        when(textMessage.getText()).thenReturn("payload");
        when(payflowEventService.updateEventFromGateWay(anyString())).thenReturn(null);

        listenerService.onMessage(textMessage, session);

        verify(jmsTemplate).send(eq("DLQ_QUEUE"), any());
    }

    @Test
    public void testNullUETR() throws Exception {
        when(textMessage.getText()).thenReturn("payload");

        EventStore store = new EventStore();
        store.setUETR(null);
        store.setPaymentStatus("Sent");

        GatewayResponseData resp = new GatewayResponseData();
        resp.setEventStore(store);

        when(payflowEventService.updateEventFromGateWay(anyString())).thenReturn(resp);

        listenerService.onMessage(textMessage, session);

        verify(jmsTemplate).send(eq("DLQ_QUEUE"), any());
    }

    @Test
    public void testErrorCode() throws Exception {
        when(textMessage.getText()).thenReturn("payload");

        EventStore store = new EventStore();
        store.setUETR("123");
        store.setPaymentStatus("Sent");

        GatewayResponseData resp = new GatewayResponseData();
        resp.setEventStore(store);
        resp.setErrorCode("ERR01");

        when(payflowEventService.updateEventFromGateWay(anyString())).thenReturn(resp);

        listenerService.onMessage(textMessage, session);

        verify(jmsTemplate).send(eq("DLQ_QUEUE"), any());
    }

    @Test
    public void testRejectedStatus() throws Exception {
        when(textMessage.getText()).thenReturn("payload");

        EventStore store = new EventStore();
        store.setUETR("123");
        store.setPaymentStatus("Rejected");

        GatewayResponseData resp = new GatewayResponseData();
        resp.setEventStore(store);

        when(payflowEventService.updateEventFromGateWay(anyString())).thenReturn(resp);

        listenerService.onMessage(textMessage, session);

        verify(jmsTemplate).send(eq("DLQ_QUEUE"), any());
    }

    @Test
    public void testInvalidMessageType() throws Exception {
        ObjectMessage objectMessage = mock(ObjectMessage.class);

        listenerService.onMessage(objectMessage, session);

        verify(jmsTemplate).send(eq("DLQ_QUEUE"), any());
    }

    @Test
    public void testExceptionHandling() throws Exception {
        when(textMessage.getText()).thenThrow(new JMSException("Boom"));

        listenerService.onMessage(textMessage, session);

        verify(jmsTemplate).send(eq("DLQ_QUEUE"), any());
    }

    @Test
    public void testDLQDisabled_NoSend() throws Exception {
        org.springframework.test.util.ReflectionTestUtils.setField(listenerService, "DLQEnabled", false);

        when(textMessage.getText()).thenThrow(new JMSException("Boom"));

        listenerService.onMessage(textMessage, session);

        verify(jmsTemplate, never()).send(eq("DLQ_QUEUE"), any());
    }
}
