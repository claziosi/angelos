echo "Copying existing truststore..."
            cp /existing-truststore/truststore.jks /merged-truststore/truststore.jks
            
            echo "Adding OpenShift CA to truststore..."
            keytool -import -trustcacerts \
              -alias openshift-service-ca \
              -file /ca-bundle/service-ca.crt \
              -keystore /merged-truststore/truststore.jks \
              -storepass changeit \
              -noprompt
            
            echo "Truststore updated successfully"
            keytool -list -keystore /merged-truststore/truststore.jks -storepass changeit
