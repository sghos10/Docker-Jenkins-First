# Docker-Jenkins-First

import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

public class PaymentRequestMapper {

    public static List<PaymentRequestMQ> mapToMQ(PaymentRequest paymentRequest) {
        List<PaymentRequestMQ> paymentRequestMQList = new ArrayList<>();

        // Check if paymentRequest or paymentDetails list is null
        if (paymentRequest == null || paymentRequest.getPaymentDetails() == null) {
            return paymentRequestMQList; // Return empty list if input is invalid
        }

        for (PaymentDetails details : paymentRequest.getPaymentDetails()) {
            if (details == null || details.getReceiver() == null || details.getPaymentInfo() == null) {
                continue; // Skip if any critical field is null
            }

            PaymentRequestMQ mqRequest = new PaymentRequestMQ();

            // Using Objects.requireNonNullElse to provide defaults if values are null
            mqRequest.setHeader(Objects.requireNonNullElse(paymentRequest.getHeader(), new Header()));
            mqRequest.setInitiator(Objects.requireNonNullElse(paymentRequest.getInitiator(), new Initiator()));
            mqRequest.setBatchInfo(Objects.requireNonNullElse(paymentRequest.getBatchInfo(), new BatchInfo()));
            mqRequest.setOriginator(Objects.requireNonNullElse(paymentRequest.getOriginator(), new Originator()));

            // Mapping receiver and payment info
            mqRequest.setReceiverAccountNo(Objects.requireNonNullElse(details.getReceiver().getAccountNo(), "UNKNOWN"));
            mqRequest.setAmount(Objects.requireNonNullElse(details.getPaymentInfo().getAmount(), 0.0)); // Default to 0.0 if null
            mqRequest.setCurrency(Objects.requireNonNullElse(details.getPaymentInfo().getCurrency(), "USD")); // Default to "USD" if null

            // Add to the list
            paymentRequestMQList.add(mqRequest);
        }

        return paymentRequestMQList;
    }
}
