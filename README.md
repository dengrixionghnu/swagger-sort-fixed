# swagger-sort-fixed
swagger在后面的版本中忽略了在ApiModelProperites中postion中排序问题，我们可以在项目中添加下面的方式来修复，重新对json属性来进行排序
```
import io.swagger.models.properties.Property;
import springfox.documentation.schema.PropertySpecification;
import springfox.documentation.service.ModelNamesRegistry;
import springfox.documentation.swagger2.mappers.ModelSpecificationMapperImpl;

import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class CustomerModelSpecificationMapper extends ModelSpecificationMapperImpl {

    @Override
    protected Map<String, Property> mapProperties(Map<String, PropertySpecification> properties, ModelNamesRegistry modelNamesRegistry) {
        Map<String, Property> treeMap = super.mapProperties(properties, modelNamesRegistry);
        List<String> keyList = properties.entrySet().stream()
                .sorted(Map.Entry.comparingByValue(
                        Comparator.comparing(PropertySpecification::getPosition)
                                .thenComparing(PropertySpecification::getName)))
                .map(Map.Entry::getKey)
                .collect(Collectors.toList());
        for (String key : keyList) {
            Property property = treeMap.get(key);
            PropertySpecification propertySpecification = properties.get(key);
            property.setPosition(propertySpecification.getPosition());
        }
        return treeMap;
    }
}
```

```
import io.swagger.models.Model;
import springfox.documentation.service.ApiListing;

import java.util.List;
import java.util.Map;


public class CustomerCompatibilityModelMapper {

    public static Map<String, Model> modelsFromApiListings(Map<String, List<ApiListing>> apiListings) {
        return new CustomerModelSpecificationMapper().modelsFromApiListings(apiListings);
    }
}

```

```
import org.aopalliance.intercept.MethodInterceptor;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.stereotype.Component;
import springfox.documentation.service.ApiListing;
import springfox.documentation.swagger2.mappers.CompatibilityModelMapper;

import java.lang.reflect.Method;
import java.util.List;
import java.util.Map;


@Component
public class ModelSpecificationMapperPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof CompatibilityModelMapper) {
            return createCompatibilityModelMapperProxy(bean);
        }
        return bean;
    }

    private Object createCompatibilityModelMapperProxy(Object bean) {
        ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.setProxyTargetClass(true);
        proxyFactory.setTargetClass(bean.getClass());
        proxyFactory.setTarget(bean);
        proxyFactory.addAdvice((MethodInterceptor) invocation -> {
            Object[] arguments = invocation.getArguments();
            Method method = invocation.getMethod();
            if ("modelsFromApiListings".equals(method.getName())) {
                return CustomerCompatibilityModelMapper.modelsFromApiListings((Map<String, List<ApiListing>>) arguments[0]);
            }
            return invocation.proceed();
        });
        return proxyFactory.getProxy();
    }
}


```
```
import io.swagger.models.Model;
import io.swagger.models.Swagger;
import io.swagger.models.properties.Property;
import org.springframework.stereotype.Component;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.swagger2.web.SwaggerTransformationContext;
import springfox.documentation.swagger2.web.WebMvcSwaggerTransformationFilter;

import javax.servlet.http.HttpServletRequest;
import java.util.Comparator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;


@Component
public class Swagger2WebMvcSwaggerTransformationFilter implements WebMvcSwaggerTransformationFilter {

    @Override
    public boolean supports(DocumentationType delimiter) {
        return DocumentationType.SWAGGER_2.equals(delimiter);
    }

    @Override
    public Swagger transform(SwaggerTransformationContext<HttpServletRequest> context) {
        Swagger swagger = context.getSpecification();
        Map<String, Model> definitions = swagger.getDefinitions();
        for (String key : definitions.keySet()) {
            Model model = definitions.get(key);
            Map<String, Property> properties = model.getProperties();
            Map<String, Property> sortedProperties = new LinkedHashMap<>(properties.size());
            List<String> fieldNameList = properties.entrySet().stream()
                    .sorted(Map.Entry.comparingByValue(
                            Comparator.comparing(Property::getPosition)
                                    .thenComparing(Property::getName)))
                    .map(Map.Entry::getKey)
                    .collect(Collectors.toList());
            for (String fieldName : fieldNameList) {
                sortedProperties.put(fieldName, properties.get(fieldName));
            }
            model.getProperties().clear();
            model.setProperties(sortedProperties);
        }
        return swagger;
    }
}


```
