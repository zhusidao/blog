# spring security oauth2源码分析

本篇只介绍授权码模式相关源码分析，本篇是基于jwt模式下进行的源码分析

TokenEndpoint#postAccessToken

```java
@RequestMapping(value = "/oauth/token", method=RequestMethod.POST)
public ResponseEntity<OAuth2AccessToken> postAccessToken(Principal principal, @RequestParam
Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {

   if (!(principal instanceof Authentication)) {
      throw new InsufficientAuthenticationException(
            "There is no client authentication. Try adding an appropriate authentication filter.");
   }
	 // 获取clientId
   String clientId = getClientId(principal);
   // 获取授权client配置信息
   ClientDetails authenticatedClient = getClientDetailsService().loadClientByClientId(clientId);
	 
   TokenRequest tokenRequest = getOAuth2RequestFactory().createTokenRequest(parameters, authenticatedClient);

   if (clientId != null && !clientId.equals("")) {
      // Only validate the client details if a client authenticated during this
      // request.
      if (!clientId.equals(tokenRequest.getClientId())) {
         // double check to make sure that the client ID in the token request is the same as that in the
         // authenticated client
         throw new InvalidClientException("Given client ID does not match authenticated client");
      }
   }
   if (authenticatedClient != null) {
      // 校验scope
      oAuth2RequestValidator.validateScope(tokenRequest, authenticatedClient);
   }
   if (!StringUtils.hasText(tokenRequest.getGrantType())) {
      throw new InvalidRequestException("Missing grant type");
   }
   if (tokenRequest.getGrantType().equals("implicit")) {
      throw new InvalidGrantException("Implicit grant type not supported from token endpoint");
   }

   if (isAuthCodeRequest(parameters)) {
      // The scope was requested or determined during the authorization step
      if (!tokenRequest.getScope().isEmpty()) {
         logger.debug("Clearing scope of incoming token request");
         tokenRequest.setScope(Collections.<String> emptySet());
      }
   }

   if (isRefreshTokenRequest(parameters)) {
      // A refresh token has its own default scopes, so we should ignore any added by the factory here.
      tokenRequest.setScope(OAuth2Utils.parseParameterList(parameters.get(OAuth2Utils.SCOPE)));
   }
	 // 获取token
   OAuth2AccessToken token = getTokenGranter().grant(tokenRequest.getGrantType(), tokenRequest);
   if (token == null) {
      throw new UnsupportedGrantTypeException("Unsupported grant type: " + tokenRequest.getGrantType());
   }

   return getResponse(token);

}
```

AuthorizationServerEndpointsConfigurer#tokenGranter

```java
private TokenGranter tokenGranter() {
   if (tokenGranter == null) {
      tokenGranter = new TokenGranter() {
         private CompositeTokenGranter delegate;

         @Override
         public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
            if (delegate == null) {
               delegate = new CompositeTokenGranter(getDefaultTokenGranters());
            }
            return delegate.grant(grantType, tokenRequest);
         }
      };
   }
   return tokenGranter;
}
```

AuthorizationServerEndpointsConfigurer#getDefaultTokenGranters

```java
private List<TokenGranter> getDefaultTokenGranters() {
   ClientDetailsService clientDetails = clientDetailsService();
   AuthorizationServerTokenServices tokenServices = tokenServices();
   AuthorizationCodeServices authorizationCodeServices = authorizationCodeServices();
   OAuth2RequestFactory requestFactory = requestFactory();

   List<TokenGranter> tokenGranters = new ArrayList<TokenGranter>();
   tokenGranters.add(new AuthorizationCodeTokenGranter(tokenServices, authorizationCodeServices, clientDetails,
         requestFactory));
   tokenGranters.add(new RefreshTokenGranter(tokenServices, clientDetails, requestFactory));
   ImplicitTokenGranter implicit = new ImplicitTokenGranter(tokenServices, clientDetails, requestFactory);
   tokenGranters.add(implicit);	
   tokenGranters.add(new ClientCredentialsTokenGranter(tokenServices, clientDetails, requestFactory));
   if (authenticationManager != null) {
      tokenGranters.add(new ResourceOwnerPasswordTokenGranter(authenticationManager, tokenServices,
            clientDetails, requestFactory));
   }
   return tokenGranters;
}
```

CompositeTokenGranter#grant

```java
public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
   for (TokenGranter granter : tokenGranters) {
      /*
       * ResourceOwnerPasswordTokenGranter     		password密码模式
       * AuthorizationCodeTokenGranter            authorization_code授权码模式
       * ClientCredentialsTokenGranter            client_credentials客户端模式
       * ImplicitTokenGranter                     implicit简化模式
       * RefreshTokenGranter                      refresh_token 刷新token
       */
      OAuth2AccessToken grant = granter.grant(grantType, tokenRequest);
      if (grant!=null) {
         return grant;
      }
   }
   return null;
}
```

AuthorizationCodeTokenGranter#getOAuth2Authentication

```java
@Override
protected OAuth2Authentication getOAuth2Authentication(ClientDetails client, TokenRequest tokenRequest) {

   Map<String, String> parameters = tokenRequest.getRequestParameters();
   // code
   String authorizationCode = parameters.get("code");
   // redirect_uri
   String redirectUri = parameters.get(OAuth2Utils.REDIRECT_URI);

   if (authorizationCode == null) {
      throw new InvalidRequestException("An authorization code must be supplied.");
   }
	 // 获取授权信息，默认为InMemoryAuthorizationCodeServices
   OAuth2Authentication storedAuth = authorizationCodeServices.consumeAuthorizationCode(authorizationCode);
   if (storedAuth == null) {
      throw new InvalidGrantException("Invalid authorization code: " + authorizationCode);
   }

   OAuth2Request pendingOAuth2Request = storedAuth.getOAuth2Request();
   // https://jira.springsource.org/browse/SECOAUTH-333
   // This might be null, if the authorization was done without the redirect_uri parameter
   // 可能为空， 如果认证的时候没有redirect_uri这个参数
   String redirectUriApprovalParameter = pendingOAuth2Request.getRequestParameters().get(
         OAuth2Utils.REDIRECT_URI);

   if ((redirectUri != null || redirectUriApprovalParameter != null)
         && !pendingOAuth2Request.getRedirectUri().equals(redirectUri)) {
      throw new RedirectMismatchException("Redirect URI mismatch.");
   }
   // 客户端id
   String pendingClientId = pendingOAuth2Request.getClientId();
   String clientId = tokenRequest.getClientId();
   if (clientId != null && !clientId.equals(pendingClientId)) {
      // just a sanity check.
      throw new InvalidClientException("Client ID mismatch");
   }

   // Secret is not required in the authorization request, so it won't be available
   // in the pendingAuthorizationRequest. We do want to check that a secret is provided
   // in the token request, but that happens elsewhere.

   Map<String, String> combinedParameters = new HashMap<String, String>(pendingOAuth2Request
         .getRequestParameters());
   // Combine the parameters adding the new ones last so they override if there are any clashes
   combinedParameters.putAll(parameters);
   
   // Make a new stored request with the combined parameters
   OAuth2Request finalStoredOAuth2Request = pendingOAuth2Request.createOAuth2Request(combinedParameters);
   
   Authentication userAuth = storedAuth.getUserAuthentication();
   
   return new OAuth2Authentication(finalStoredOAuth2Request, userAuth);

}
```



接下来为refresh_token刷新access_token的逻辑分析

RefreshTokenGranter#getAccessToken

```java
@Override
protected OAuth2AccessToken getAccessToken(ClientDetails client, TokenRequest tokenRequest) {
   String refreshToken = tokenRequest.getRequestParameters().get("refresh_token");
   return getTokenServices().refreshAccessToken(refreshToken, tokenRequest);
}
```

DefaultTokenServices#refreshAccessToken

```java
@Transactional(noRollbackFor={InvalidTokenException.class, InvalidGrantException.class})
public OAuth2AccessToken refreshAccessToken(String refreshTokenValue, TokenRequest tokenRequest)
      throws AuthenticationException {

   if (!supportRefreshToken) {
      throw new InvalidGrantException("Invalid refresh token: " + refreshTokenValue);
   }
	 // 读取refreshToken
   OAuth2RefreshToken refreshToken = tokenStore.readRefreshToken(refreshTokenValue);
   if (refreshToken == null) {
      throw new InvalidGrantException("Invalid refresh token: " + refreshTokenValue);
   }
	 // 读取认证信息
   OAuth2Authentication authentication = tokenStore.readAuthenticationForRefreshToken(refreshToken);
   if (this.authenticationManager != null && !authentication.isClientOnly()) {
      // The client has already been authenticated, but the user authentication might be old now, so give it a
      // chance to re-authenticate.
      Authentication user = new PreAuthenticatedAuthenticationToken(authentication.getUserAuthentication(), "", authentication.getAuthorities());
      user = authenticationManager.authenticate(user);
      Object details = authentication.getDetails();
      authentication = new OAuth2Authentication(authentication.getOAuth2Request(), user);
      authentication.setDetails(details);
   }
   String clientId = authentication.getOAuth2Request().getClientId();
   if (clientId == null || !clientId.equals(tokenRequest.getClientId())) {
      throw new InvalidGrantException("Wrong client for this refresh token: " + refreshTokenValue);
   }

   // clear out any access tokens already associated with the refresh
   // token.
   // 使用refreshToken清空accessToken
   // jwt没有做处理
   tokenStore.removeAccessTokenUsingRefreshToken(refreshToken);

   if (isExpired(refreshToken)) {
      tokenStore.removeRefreshToken(refreshToken);
      throw new InvalidTokenException("Invalid refresh token (expired): " + refreshToken);
   }
   authentication = createRefreshedAuthentication(authentication, tokenRequest);
 
   if (!reuseRefreshToken) {
      // refresh_token不重复利用
      // 移除refresh_token
      tokenStore.removeRefreshToken(refreshToken);
      refreshToken = createRefreshToken(authentication);
   }
   // 创建access_token
   OAuth2AccessToken accessToken = createAccessToken(authentication, refreshToken);
   tokenStore.storeAccessToken(accessToken, authentication);
   if (!reuseRefreshToken) {
      tokenStore.storeRefreshToken(accessToken.getRefreshToken(), authentication);
   }
   return accessToken;
}
```

> jwt对新的token不做存在，对旧的access_token也不做处理（参考access_token的无状态特性）

JwtTokenStore#readRefreshToken

```java
@Override
public OAuth2RefreshToken readRefreshToken(String tokenValue) {
   // 解析tokenValue的值并转化为OAuth2AccessToken对象
   OAuth2AccessToken encodedRefreshToken = convertAccessToken(tokenValue);
   // 过期时间如果不为空，就会创建一个DefaultExpiringOAuth2RefreshToken对象
   OAuth2RefreshToken refreshToken = createRefreshToken(encodedRefreshToken);
   if (approvalStore != null) {
      // 读取授权信息
      OAuth2Authentication authentication = readAuthentication(tokenValue);
      if (authentication.getUserAuthentication() != null) {
         // 用户名
         String userId = authentication.getUserAuthentication().getName();
         // clientId
         String clientId = authentication.getOAuth2Request().getClientId();
         // 获取同意的认证信息
         Collection<Approval> approvals = approvalStore.getApprovals(userId, clientId);
         Collection<String> approvedScopes = new HashSet<String>();
         for (Approval approval : approvals) {
            if (approval.isApproved()) {
               approvedScopes.add(approval.getScope());
            }
         }
         if (!approvedScopes.containsAll(authentication.getOAuth2Request().getScope())) {
            // 同意的认证信息没有全部包含该请求的scope
            return null;
         }
      }
   }
   return refreshToken;
}
```



**JwtTokenStore#readRefreshToken**

```java
private OAuth2AccessToken convertAccessToken(String tokenValue) {
   // decode方法会将refresh_token解析并转化为map
   return jwtTokenEnhancer.extractAccessToken(tokenValue, jwtTokenEnhancer.decode(tokenValue));
}
```

> jwtTokenEnhancer.decode解析出来后的值
>
> "user_name" -> "user1"
> "scope" -> {ArrayList@9464}  size = 1
> "ati" -> "e6703af4-bc6c-4266-8de3-c7f99f243a93"
> "exp" -> {Long@9466} 1607494348
> "authorities" -> {ArrayList@9467}  size = 1
> "jti" -> "fb5ffe13-ac8a-4c52-896e-075287a9d13f"
> "client_id" -> "messaging-client"

DefaultAccessTokenConverter#extractAccessToken

```java
public OAuth2AccessToken extractAccessToken(String value, Map<String, ?> map) {
   DefaultOAuth2AccessToken token = new DefaultOAuth2AccessToken(value);
   Map<String, Object> info = new HashMap<String, Object>(map);
   info.remove(EXP);
   info.remove(AUD);
   info.remove(clientIdAttribute);
   info.remove(scopeAttribute);
   if (map.containsKey(EXP)) {
      // 过期时间*1000转化为毫米
      token.setExpiration(new Date((Long) map.get(EXP) * 1000L));
   }
   if (map.containsKey(JTI)) {
      info.put(JTI, map.get(JTI));
   }
   // scope值
   token.setScope(extractScope(map));
   token.setAdditionalInformation(info);
   return token;
}
```

JwtTokenStore#createRefreshToken

```java
private OAuth2RefreshToken createRefreshToken(OAuth2AccessToken encodedRefreshToken) {
   if (!jwtTokenEnhancer.isRefreshToken(encodedRefreshToken)) {
      // 不是refresh token就抛出异常，refresh_token解析出来后会有一个ati属性
      throw new InvalidTokenException("Encoded token is not a refresh token");
   }
   if (encodedRefreshToken.getExpiration()!=null) {
      // 过期时间不为空
      return new DefaultExpiringOAuth2RefreshToken(encodedRefreshToken.getValue(),
            encodedRefreshToken.getExpiration());        
   }
   return new DefaultOAuth2RefreshToken(encodedRefreshToken.getValue());
}
```



**JwtTokenStore#readAuthentication**

```java
@Override
public OAuth2Authentication readAuthentication(String token) {
   return jwtTokenEnhancer.extractAuthentication(jwtTokenEnhancer.decode(token));
}
```

JwtAccessTokenConverter#extractAuthentication

```java
@Override
public OAuth2Authentication extractAuthentication(Map<String, ?> map) {
   return tokenConverter.extractAuthentication(map);
}
```

DefaultAccessTokenConverter#extractAuthentication

```java
public OAuth2Authentication extractAuthentication(Map<String, ?> map) {
   Map<String, String> parameters = new HashMap<String, String>();
   // 提取scope
   Set<String> scope = extractScope(map);
   // 提取权限信息
   Authentication user = userTokenConverter.extractAuthentication(map);
   // client_id
   String clientId = (String) map.get(clientIdAttribute);
   parameters.put(clientIdAttribute, clientId);
   // includeGrantType默认为false
   if (includeGrantType && map.containsKey(GRANT_TYPE)) {
      parameters.put(GRANT_TYPE, (String) map.get(GRANT_TYPE));
   }
   // 获取aud属性
   Set<String> resourceIds = new LinkedHashSet<String>(map.containsKey(AUD) ? getAudience(map)
         : Collections.<String>emptySet());
   
   Collection<? extends GrantedAuthority> authorities = null;
   if (user==null && map.containsKey(AUTHORITIES)) {
      @SuppressWarnings("unchecked")
      String[] roles = ((Collection<String>)map.get(AUTHORITIES)).toArray(new String[0]);
      authorities = AuthorityUtils.createAuthorityList(roles);
   }
   OAuth2Request request = new OAuth2Request(parameters, clientId, authorities, true, scope, resourceIds, null, null,
         null);
   return new OAuth2Authentication(request, user);
}
```

DefaultUserAuthenticationConverter#extractAuthentication

```java
public Authentication extractAuthentication(Map<String, ?> map) {
   if (map.containsKey(USERNAME)) {
      // 获取用户名
      Object principal = map.get(USERNAME);
      // 获取权限
      Collection<? extends GrantedAuthority> authorities = getAuthorities(map);
      if (userDetailsService != null) {
         // 如果userDetailsService不为空，会调用loadUserByUsername去获取用户名和权限
         UserDetails user = userDetailsService.loadUserByUsername((String) map.get(USERNAME));
         authorities = user.getAuthorities();
         principal = user;
      }
      return new UsernamePasswordAuthenticationToken(principal, "N/A", authorities);
   }
   return null;
}
```



**JwtTokenStore#readAuthenticationForRefreshToken**

```java
@Override
public OAuth2Authentication readAuthenticationForRefreshToken(OAuth2RefreshToken token) {
   // 该方法上文已经有分析
   return readAuthentication(token.getValue());
}
```



熟悉的**ProviderManager#authenticate**

```java
public Authentication authenticate(Authentication authentication)
      throws AuthenticationException {
   Class<? extends Authentication> toTest = authentication.getClass();
   AuthenticationException lastException = null;
   AuthenticationException parentException = null;
   Authentication result = null;
   Authentication parentResult = null;
   boolean debug = logger.isDebugEnabled();

   for (AuthenticationProvider provider : getProviders()) {
      // getProviders()只有一个PreAuthenticatedAuthenticationProvider
      if (!provider.supports(toTest)) {
         continue;
      }

      if (debug) {
         logger.debug("Authentication attempt using "
               + provider.getClass().getName());
      }

      try {
         result = provider.authenticate(authentication);

         if (result != null) {
            copyDetails(authentication, result);
            break;
         }
      }
      catch (AccountStatusException | InternalAuthenticationServiceException e) {
         prepareException(e, authentication);
         // SEC-546: Avoid polling additional providers if auth failure is due to
         // invalid account status
         throw e;
      } catch (AuthenticationException e) {
         lastException = e;
      }
   }

   if (result == null && parent != null) {
      // Allow the parent to try.
      try {
         result = parentResult = parent.authenticate(authentication);
      }
      catch (ProviderNotFoundException e) {
         // ignore as we will throw below if no other exception occurred prior to
         // calling parent and the parent
         // may throw ProviderNotFound even though a provider in the child already
         // handled the request
      }
      catch (AuthenticationException e) {
         lastException = parentException = e;
      }
   }

   if (result != null) {
      if (eraseCredentialsAfterAuthentication
            && (result instanceof CredentialsContainer)) {
         // Authentication is complete. Remove credentials and other secret data
         // from authentication
         ((CredentialsContainer) result).eraseCredentials();
      }

      // If the parent AuthenticationManager was attempted and successful than it will publish an AuthenticationSuccessEvent
      // This check prevents a duplicate AuthenticationSuccessEvent if the parent AuthenticationManager already published it
      if (parentResult == null) {
         eventPublisher.publishAuthenticationSuccess(result);
      }
      return result;
   }

   // Parent was null, or didn't authenticate (or throw an exception).

   if (lastException == null) {
      lastException = new ProviderNotFoundException(messages.getMessage(
            "ProviderManager.providerNotFound",
            new Object[] { toTest.getName() },
            "No AuthenticationProvider found for {0}"));
   }

   // If the parent AuthenticationManager was attempted and failed than it will publish an AbstractAuthenticationFailureEvent
   // This check prevents a duplicate AbstractAuthenticationFailureEvent if the parent AuthenticationManager already published it
   if (parentException == null) {
      prepareException(lastException, authentication);
   }

   throw lastException;
}
```

PreAuthenticatedAuthenticationProvider#authenticate

```java
public Authentication authenticate(Authentication authentication)
      throws AuthenticationException {
   if (!supports(authentication.getClass())) {
      return null;
   }
	 
   if (logger.isDebugEnabled()) {
      logger.debug("PreAuthenticated authentication request: " + authentication);
   }
   // 用户名校验
   if (authentication.getPrincipal() == null) {
      logger.debug("No pre-authenticated principal found in request.");

      if (throwExceptionWhenTokenRejected) {
         throw new BadCredentialsException(
               "No pre-authenticated principal found in request.");
      }
      return null;
   }
   // 密码校验
   if (authentication.getCredentials() == null) {
      logger.debug("No pre-authenticated credentials found in request.");

      if (throwExceptionWhenTokenRejected) {
         throw new BadCredentialsException(
               "No pre-authenticated credentials found in request.");
      }
      return null;
   }

   UserDetails ud = preAuthenticatedUserDetailsService
         .loadUserDetails((PreAuthenticatedAuthenticationToken) authentication);

   userDetailsChecker.check(ud);

   PreAuthenticatedAuthenticationToken result = new PreAuthenticatedAuthenticationToken(
         ud, authentication.getCredentials(), ud.getAuthorities());
   result.setDetails(authentication.getDetails());

   return result;
}
```



DefaultTokenServices#createAccessToken

```java
private OAuth2AccessToken createAccessToken(OAuth2Authentication authentication, OAuth2RefreshToken refreshToken) {
   // 创建一个token
   DefaultOAuth2AccessToken token = new DefaultOAuth2AccessToken(UUID.randomUUID().toString());
   // 有效时间（单位秒）
   int validitySeconds = getAccessTokenValiditySeconds(authentication.getOAuth2Request());
   if (validitySeconds > 0) {
      token.setExpiration(new Date(System.currentTimeMillis() + (validitySeconds * 1000L)));
   }
   token.setRefreshToken(refreshToken);
   token.setScope(authentication.getOAuth2Request().getScope());

   return accessTokenEnhancer != null ? accessTokenEnhancer.enhance(token, authentication) : token;
}
```

JwtAccessTokenConverter#enhance

```java
public OAuth2AccessToken enhance(OAuth2AccessToken accessToken, OAuth2Authentication authentication) {
   DefaultOAuth2AccessToken result = new DefaultOAuth2AccessToken(accessToken);
   Map<String, Object> info = new LinkedHashMap<String, Object>(accessToken.getAdditionalInformation());
   String tokenId = result.getValue();
   if (!info.containsKey(TOKEN_ID)) {
      // key为jti，value为token，使用access_token的jti的值
      info.put(TOKEN_ID, tokenId);
   }
   else {
      tokenId = (String) info.get(TOKEN_ID);
   }
   // 添加附加信息
   result.setAdditionalInformation(info);
   // 产生新的jwt字符串
   result.setValue(encode(result, authentication));
   // 获取refreshToken
   OAuth2RefreshToken refreshToken = result.getRefreshToken();
   if (refreshToken != null) {
      DefaultOAuth2AccessToken encodedRefreshToken = new DefaultOAuth2AccessToken(accessToken);
      encodedRefreshToken.setValue(refreshToken.getValue());
      // Refresh tokens do not expire unless explicitly of the right type
      encodedRefreshToken.setExpiration(null);
      try {
         // 解析refreshToken中的claim
         Map<String, Object> claims = objectMapper
               .parseMap(JwtHelper.decode(refreshToken.getValue()).getClaims());
         if (claims.containsKey(TOKEN_ID)) {
            // 旧的token
            encodedRefreshToken.setValue(claims.get(TOKEN_ID).toString());
         }
      }
      catch (IllegalArgumentException e) {
      }
      Map<String, Object> refreshTokenInfo = new LinkedHashMap<String, Object>(
            accessToken.getAdditionalInformation());
      // jti使用旧refresh的jti
      refreshTokenInfo.put(TOKEN_ID, encodedRefreshToken.getValue());
      // ati使用新创建的access_token的TOKEN_ID
      refreshTokenInfo.put(ACCESS_TOKEN_ID, tokenId);
      // 附加信息
      encodedRefreshToken.setAdditionalInformation(refreshTokenInfo);
      // 产生新的refresh_token
      DefaultOAuth2RefreshToken token = new DefaultOAuth2RefreshToken(
            encode(encodedRefreshToken, authentication));
      if (refreshToken instanceof ExpiringOAuth2RefreshToken) {
         // 过期时间
         Date expiration = ((ExpiringOAuth2RefreshToken) refreshToken).getExpiration();
         encodedRefreshToken.setExpiration(expiration);
         // 重新进行编码产生新的refresh_token
         token = new DefaultExpiringOAuth2RefreshToken(encode(encodedRefreshToken, authentication), expiration);
      }
      result.setRefreshToken(token);
   }
   return result;
}
```

DefaultAccessTokenConverter#convertAccessToken

```java
public Map<String, ?> convertAccessToken(OAuth2AccessToken token, OAuth2Authentication authentication) {
   Map<String, Object> response = new HashMap<String, Object>();
   OAuth2Request clientToken = authentication.getOAuth2Request();

   if (!authentication.isClientOnly()) {
      // 用户名、权限信息
      response.putAll(userTokenConverter.convertUserAuthentication(authentication.getUserAuthentication()));
   } else {
      if (clientToken.getAuthorities()!=null && !clientToken.getAuthorities().isEmpty()) {
         response.put(UserAuthenticationConverter.AUTHORITIES,
                   AuthorityUtils.authorityListToSet(clientToken.getAuthorities()));
      }
   }

   if (token.getScope()!=null) {
      // scope
      response.put(scopeAttribute, token.getScope());
   }
   if (token.getAdditionalInformation().containsKey(JTI)) {
      // jti
      response.put(JTI, token.getAdditionalInformation().get(JTI));
   }

   if (token.getExpiration() != null) {
      // 过期时间
      response.put(EXP, token.getExpiration().getTime() / 1000);
   }
   // includeGrantType默认为false
   if (includeGrantType && authentication.getOAuth2Request().getGrantType()!=null) {
      // grant_type
      response.put(GRANT_TYPE, authentication.getOAuth2Request().getGrantType());
   }

   response.putAll(token.getAdditionalInformation());
   // client_id
   response.put(clientIdAttribute, clientToken.getClientId());
   if (clientToken.getResourceIds() != null && !clientToken.getResourceIds().isEmpty()) {
      // aud
      response.put(AUD, clientToken.getResourceIds());
   }
   return response;
}
```