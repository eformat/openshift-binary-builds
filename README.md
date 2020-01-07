# openshift-binary-builds

This drove me a little crazy on OCP 4.X

### Binary builds using an authenticated registry

```
oc new-project test
oc create secret generic imagestreamsecret --from-literal=.dockerconfigjson=`oc get secret samples-registry-credentials -n openshift -o jsonpath="{.data['\.dockerconfigjson']}" |base64 -d` --type=kubernetes.io/dockerconfigjson
oc secrets link default imagestreamsecret --for=pull
oc secrets link builder imagestreamsecret
oc new-build --name=test --dockerfile=$'FROM registry.redhat.io/ubi8/ubi:latest\nCMD ["sleep", "infinity"]'
oc new-app test
```

```
oc new-project test
oc create secret generic imagestreamsecret --from-literal=.dockerconfigjson=`oc get secret samples-registry-credentials -n openshift -o jsonpath="{.data['\.dockerconfigjson']}" |base64 -d` --type=kubernetes.io/dockerconfigjson
oc new-build --binary --name=test --strategy=docker
oc set build-secret --pull bc/test imagestreamsecret
cat <<EOF > Dockerfile
FROM registry.redhat.io/ubi8/ubi:latest
CMD ["sleep", "infinity"]
EOF
oc start-build test --from-dir=. --follow
oc new-app test
```

### With Entitlements

```
mkdir -p etc-pki-entitlement rhsm-sa rhsm-conf
scp entitled-host:/etc/pki/entitlement/2007150637850756092.pem etc-pki-entitlement/
scp entitled-host:/etc/pki/entitlement/2007150637850756092-key.pem etc-pki-entitlement/
scp entitled-host:/etc/rhsm/ca/redhat-uep.pem rhsm-sa/
scp entitled-host:/etc/rhsm/rhsm.conf rhsm-conf/
```

```
cat <<EOF > Dockerfile
FROM registry.redhat.io/ubi8/ubi:latest
USER root
# Copy entitlements
COPY ./etc-pki-entitlement /etc/pki/entitlement
# Copy subscription manager configurations
COPY ./rhsm-conf /etc/rhsm
COPY ./rhsm-ca /etc/rhsm/ca
# Delete /etc/rhsm-host to use entitlements from the build container
# Initialize /etc/yum.repos.d/redhat.repo
# See https://access.redhat.com/solutions/1443553
RUN rm -f /etc/rhsm-host && \
    yum repolist --disablerepo=* && \
    subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms  && \
    yum -y update && \
    yum -y install httpd && \
    rm -rf /var/cache/yum && \
    rm -rf /etc/pki/entitlement && \
    rm -rf /etc/rhsm
# Add default Web page and expose port
RUN echo "The Web Server is Running" > /var/www/html/index.html
EXPOSE 80
# Start the service
CMD ["-D", "FOREGROUND"]
ENTRYPOINT ["/usr/sbin/httpd"]
EOF

oc new-project test
oc create secret generic imagestreamsecret --from-literal=.dockerconfigjson=`oc get secret samples-registry-credentials -n openshift -o jsonpath="{.data['\.dockerconfigjson']}" |base64 -d` --type=kubernetes.io/dockerconfigjson
oc new-build --binary --name=test
oc set build-secret --pull bc/test imagestreamsecret
oc start-build test --from-dir=. --follow
oc adm policy add-scc-to-user anyuid -z default -n test
oc new-app test
oc expose svc test
curl $(oc get route test --template='{{ .spec.host }}')
```