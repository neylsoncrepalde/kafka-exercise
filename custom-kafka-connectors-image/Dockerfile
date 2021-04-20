FROM confluentinc/cp-kafka-connect-base

RUN confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:10.1.1 \
    && confluent-hub install --no-prompt confluentinc/kafka-connect-s3:10.0.0