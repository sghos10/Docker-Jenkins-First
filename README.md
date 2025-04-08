# Docker-Jenkins-First
 if (input == null || input.getPaymentDetails() == null ||
            input.getOriginator() == null || input.getBatchInfo() == null) {
            System.err.println("Invalid input JSON: Missing required sections.");
            return;
        }

        List<FinalPaymentWrapper> outputList = new ArrayList<>();

        for (PaymentDetail detail : input.getPaymentDetails()) {
            if (detail == null || detail.getReceiver() == null || detail.getPaymentInfo() == null) {
                System.err.println("Skipping invalid payment detail.");
                continue;
            }

            Receiver receiver = detail.getReceiver();
            PaymentInfo info = detail.getPaymentInfo();

            FinalPaymentWrapper wrapper = new FinalPaymentWrapper();

            // participantInfo
            ParticipantInfo participantInfo = new ParticipantInfo();
            participantInfo.setAccountNumber(
                Objects.requireNonNullElse(input.getOriginator().getAccountNumber(), "UNKNOWN_ACC")
            );
            participantInfo.setCifStaticKey("123456"); // Static for now

            // recipientInfo
            RecipientInfo recipientInfo = new RecipientInfo();
            recipientInfo.setRecipientFirstOrBusName(
                Objects.requireNonNullElse(receiver.getAccountName(), "UNKNOWN")
            );
            recipientInfo.setRecipientLastName(
                Objects.requireNonNullElse(receiver.getAccountLastName(), "")
            );
            recipientInfo.setRecipientToken(
                Objects.requireNonNullElse(receiver.getAccountNumber(), "")
            );

            // paymentInfo
            PaymentInfoOutput paymentInfo = new PaymentInfoOutput();
            paymentInfo.setAmount(info.getAmount());
            paymentInfo.setMemo(
                Objects.requireNonNullElse(info.getPurpose(), "")
            );

            // paymentRequest
            PaymentRequest paymentRequest = new PaymentRequest();
            paymentRequest.setParticipantInfo(participantInfo);
            paymentRequest.setRecipientInfo(recipientInfo);
            paymentRequest.setPaymentInfo(paymentInfo);

            // metaData
            MetaData metaData = new MetaData();
            metaData.setCustomerRefNumber(
                Objects.requireNonNullElse(info.getEndToEndId(), "UNKNOWN_REF")
            );
            metaData.setBatchId(
                Objects.requireNonNullElse(input.getBatchInfo().getBatchId(), "UNKNOWN_BATCH")
            );
            metaData.setUetr(
                Objects.requireNonNullElse(info.getUetr(), "UNKNOWN_UETR")
            );

            // Final wrapper
            wrapper.setPaymentRequest(paymentRequest);
            wrapper.setMetaData(metaData);
            outputList.add(wrapper);

