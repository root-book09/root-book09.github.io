---
title: Rancher SSO연동
author: root-book09
date: 2023-01-19
layout: post
---

#rancher #sso 

# 1. Rancher란
<hr>
One plactform for Kubernetes Management = 쿠버네티스 관리 플랫폼

# 2. SSO 연결
<hr>
### 1. 조건
- 폐쇄망
- Rancher version 2.6x
- 같은 폐쇄망에 구성된 다른 서버에 올라가 있는 Rancher 화면을 SSO로 연결, 로그인 화면이 아닌 첫 화면으로 연결

### 2. Postman API TEST

> [POST] https://{server-url}/v3-public/localProviders/local?action=login
> Body : JSON 
> 		{ "username": "{username}", "password": "{password}" }

해당 요청 시, 결과로 토큰이 포함된 JSON 형태의 값을 받을 수 있음

### 3. Rancher Login Cookies 확인
-  `R_SESS`라는 이름으로 로그인이 될 때마다 토큰이 생성됨을 확인
-  해당 토큰은 Rancher > admin login > user icon click > Account&API Keys에서 확인 할 수 있음. login시 생성되고, logout또는 시간 만료 시 삭제됨

### 4. Java 소스 구현

```java
/*
@Data
public static Class SiteInfo {
	private String serverUrl;
	private String authPath;
	private String username;
	private String userpass;
	private String cookieName;
	private String cookieDomain;
}
*/
public Map<String, Object> getSSOInfo(final Siteinfo siteInfo){
	final HttpHeaders headers = new HttpHeaders();
	final Map<String, String> valueMap = new LinkedHashMap<>();
	valueMap.put("username", siteInfo.getUsername()); //  사용자 ID
	valueMap.put("password", siteInfo.getUserpass()); // 사용자 PW

	final HttpEntity<Map<String, String>> httpentity = new HttpEntity<>(valueMap, headers);
	final String authUrl = siteInfo.getServerUrl()+siteInfo.getAuthPath();
	// https://{server-url}/v3-public/localProviders/local?action=login
	final boolean isSecure = this.isSequre(authUrl);

	final ResponseEntity<String> res = this.getRestTemplate(isSequre).exchanage(authUrl, HttpMethod.POST, httpEntity, String.class);
	final String resBody = res.getBody();

	final JSONObject jsonObject  = new JSONObject(resBody);

	String token = "";
	if(jsonObject != null){
		token = jsonObject.getString("token");
		log.info("RANCHER TOKEN [{}]", token);
	}

	final HttpServletResponse response = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getResponse();
	if(StringUtils.isNotBlank(token)){
		// cookieName = "R_SESS" , coockieDomain = "{sub-domain}"
		// 쿠키값을 생성하고, 그 쿠키를 읽어들이기 위해서는 sub domain을 일치시켜줘야한다. 
		final HttpCookie httpCookie = ResponseCookie.from(siteInfo.getCookieName(), token).httpOnly(true).maxAge(3600).domain(siteInfo.getCookieDomain()).path("/").source(true).build();
		response.addHeaer(HttpHeaders.SET_COOCIE, httpCookie.toString());

		return this.getResultMap(SSOResultType.SUCCESS, siteInfo.getServerUrl());
	}
	return this.getResultMap(SSOResultType.RANCHER_LOGIN_FAIL, null);
}

private boolean isSequre(final String url){
	return StringUtils.startsWithIgnoreCase(url, "https");
}

private RestTemplate getRestTemplate(final boolen isSecure){

	final HttpComponentsClientHttpRequestFactory httpClientFactory = new HttpComponentsClientHttprequestFactory();
	if(isSequre){
		final SSLContext sslContext;
		try{
			sslContext = SSLContexts.custom().leadTrustMeterial(null, new TrustSelfSignedStrategy()).build();
			final SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(sslContext, new HostnameVerifier(){
				@Override
				public boolen verify(final String hostname, final SSLSeesion session){
					return true;
				}
			});
			httpClientFactory.setHttpsClient(HttpClients.custom().setSSLSocketFacotyr(sslsf).build())
		}catch(KeyManagementException | NoSuchAlgorithmException | KeyStoreException e){
			log.error("### SSLContext ERROR :: {}", e.getMessage(), e)
		}
	}

	final RestTemplate restTemplate = new RestTemplate(httpsClientFactory);
	restTemplate.getMessageConverters().add(new MappingJacson2HttpMessageConverter());
	restTemplate.getMessageConverters().add(new StringHttpMessgeConverter());
	
	return restTemplate;
}

```

