import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.accept.ContentNegotiationManager;
import org.springframework.web.accept.HeaderContentNegotiationStrategy;
import org.springframework.web.accept.ParameterContentNegotiationStrategy;
import org.springframework.web.accept.ContentNegotiationStrategy;

import java.util.*;

@Configuration
public class WebMvcConfig {

    @Bean
    public ContentNegotiationManager mvcContentNegotiationManager() {
        Map<String, MediaType> mediaTypes = new HashMap<>();
        mediaTypes.put("json", MediaType.APPLICATION_JSON);
        mediaTypes.put("xml", MediaType.APPLICATION_XML);
        // Note: do NOT include "yaml" here

        List<ContentNegotiationStrategy> strategies = new ArrayList<>();
        strategies.add(new ParameterContentNegotiationStrategy(mediaTypes));
        strategies.add(new HeaderContentNegotiationStrategy());

        return new ContentNegotiationManager(strategies);
    }
}

