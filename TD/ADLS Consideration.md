package com.example.enterprise_kafka_starter.config;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ContainerProperties;

import java.util.HashMap;
import java.util.Map;

@AutoConfiguration
@ConditionalOnClass(ConcurrentKafkaListenerContainerFactory.class)
@EnableConfigurationProperties(EnterpriseKafkaProperties.class)
public class EnterpriseKafkaAutoConfiguration {

    /**
     * Build all the SASL_SSL + OAuthBearer + truststore properties
     * from the high-level values teams put under enterprise.kafka.oauth.*
     */
    @Bean
    @ConditionalOnMissingBean(name = "enterpriseKafkaCommonProps")
    Map<String, Object> enterpriseKafkaCommonProps(EnterpriseKafkaProperties ek) {
        OauthKafkaProperties p = ek.getOauth();
        Map<String, Object> props = new HashMap<>();

        // Core security
        props.put("security.protocol", "SASL_SSL");
        props.put("sasl.mechanism", "OAUTHBEARER");
        props.put("sasl.login.callback.handler.class",
                "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler");

        // SSL
        props.put("ssl.truststore.location", p.getSslTruststoreLocation());
        props.put("ssl.truststore.password", p.getSslTruststorePassword());
        props.put("ssl.truststore.type", p.getSslTruststoreType());

        // OAuth specific
        props.put("sasl.oauthbearer.token.endpoint.url", p.getTokenEndpointUrl());
        props.put("sasl.oauthbearer.sub.claim.name", p.getSubClaimName());

        // JAAS from the 9–10 values they provide
        props.put("sasl.jaas.config", buildJaasConfig(p));

        return props;
    }

    private String buildJaasConfig(OauthKafkaProperties p) {
        return String.format(
                "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required " +
                "clientId=\"%s\" " +
                "clientSecret=\"%s\" " +
                "scope=\"%s\" " +
                "extension_logicalCluster=\"%s\" " +
                "extension_identityPoolId=\"%s\" " +
                "oauthbearer.token.endpoint.url=\"%s\";",
                p.getClientId(),
                p.getClientSecret(),
                p.getScope(),
                p.getKafkaClusterId(),
                p.getIdentityPoolId(),
                p.getTokenEndpointUrl()
        );
    }

    @Bean(name = "enterpriseConsumerFactory")
    @ConditionalOnMissingBean(name = "enterpriseConsumerFactory")
    ConsumerFactory<Object, Object> enterpriseConsumerFactory(
            KafkaProperties bootProps,
            EnterpriseKafkaProperties enterpriseProps,
            Map<String, Object> enterpriseKafkaCommonProps) {

        Map<String, Object> props = new HashMap<>(bootProps.buildConsumerProperties(null));

        // 1) add all SASL/SSL/OAuth stuff built from enterprise.kafka.oauth.*
        props.putAll(enterpriseKafkaCommonProps);

        // 2) add tuning overrides from EnterpriseKafkaProperties (poll timeout, retries, etc.)
        props.putAll(enterpriseProps.asKafkaOverrides());

        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean(name = "enterpriseKafkaListenerContainerFactory")
    @ConditionalOnMissingBean(name = "enterpriseKafkaListenerContainerFactory")
    ConcurrentKafkaListenerContainerFactory<Object, Object> enterpriseKafkaListenerContainerFactory(
            ConsumerFactory<Object, Object> enterpriseConsumerFactory,
            EnterpriseKafkaProperties enterpriseProps) {

        ConcurrentKafkaListenerContainerFactory<Object, Object> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(enterpriseConsumerFactory);

        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        factory.getContainerProperties().setPollTimeout(enterpriseProps.getPollTimeoutMs());
        factory.setCommonErrorHandler(enterpriseProps.buildErrorHandler());

        return factory;
    }
}
