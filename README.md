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






{
  "header": {
    "channelId": "OLBMOBI",
    "channelSubId": "sample",
    "type": "COM|RET|BUS"
  },
  "initiator": {
    "id": "12345678901234567890",
    "idType": "CIF|CID",
    "name": "Truist Disbursement Corp"
  },
  "batchInfo": {
    "batchId": "TRUIST-1596471684",
    "batchAmount": 6000,
    "numberOfReceivers": 2
  },
  "originator": {
    "accountName": "Truist Zelle Payment",
    "accountNumber": "0000000010010",
    "accountType": "ZELLE"
  },
  "paymentDetails": [
    {
      "receiver": {
        "accountName": "Recepient 1",
        "accountLastName": "Lst 1",
        "accountNumber": "Recepient-1-Lst-1@gmail.com",
        "accountType": "SAV",
        "tokenType": "EMAIL|PHN"
      },
      "paymentInfo": {
        "amount": 3000,
        "currency": "USD",
        "executionDate": "2025-03-31",
        "endToEndId": "123456789987654321",
        "uetr": "7b0b8e7e-6893-4b36-93bc-c68aabaf5859",
        "purpose": "Mobile Services"
      }
    },
    {
      "receiver": {
        "accountName": "Recepient 2",
        "accountLastName": "Last 2"
        "accountNumber": "Recepient-2-Lst-2@gmail.com",
        "accountType": "ZELLE",
        "tokenType": "EMAIL|PHN"
      },
      "paymentInfo": {
        "amount": 3000,
        "currency": "USD",
        "executionDate": "2025-03-31",
        "endToEndId": "123456789987654322",
        "uetr": "e6cd77b8-b7b4-4d7c-8845-fadfee9952bd",
        "purpose": "Network Services"
      }
    }
  ]
}


{
  "paymentRequest": {
    "participantInfo": {
      "accountNumber": "0000000010010",
      "cifStaticKey": "123456"
    },
    "recipientInfo": {
      "recipientFirstOrBusName": "Recepient 1",
      "recipientLastName": "Lst 1",
      "recipientToken": "Recepient-1-Lst-1@gmail.com"
    },
    "paymentInfo": {
      "amount": 3000,
      "memo": "Mobile Services"
    }
  },
  "metaData": {
    "customerRefNumber": "123456789987654321",
    "batchId": "TRUIST-1596471684",
    "uetr": "7b0b8e7e-6893-4b36-93bc-c68aabaf5859"
  }
}

-------------------------------------------------

{
  "paymentRequest": {
    "participantInfo": {
      "accountNumber": "0000000010010",
      "cifStaticKey": "123456"
    },
    "recipientInfo": {
      "recipientFirstOrBusName": "Recepient 2",
      "recipientLastName": "Lst 2",
      "recipientToken": "Recepient-2-Lst-2@gmail.com"
    },
    "paymentInfo": {
      "amount": 3000,
      "memo": "Network Services"
    }
  },
  "metaData": {
    "customerRefNumber": "123456789987654322",
    "batchId": "TRUIST-1596471684",
    "uetr": "e6cd77b8-b7b4-4d7c-8845-fadfee9952bd"
  }
}

