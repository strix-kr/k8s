kubectl config set-credentials "k8s.strix.kr-ldap-auth" \
  --name k8s-strix-kr
  --auth-provider oidc \
  --auth-provider-arg idp-issuer-url=https://iam.strix.kr/auth/realms/default \
  --auth-provider-arg client-id=kubernetes \
  --auth-provider-arg client-secret=343deb41-052a-46fc-b4c9-27ee898f0717