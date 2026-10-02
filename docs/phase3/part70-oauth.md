# Part 70: OAuth 2.0 และ OpenID Connect

## เป้าหมายการเรียนรู้
- เข้าใจ OAuth 2.0 อย่างลึกซึ้ง
- Authorization Code Flow with PKCE
- Client Credentials Flow
- OpenID Connect (OIDC)
- JWT Validation
- Keycloak Integration
- Google OAuth
- สร้าง Custom OAuth Server

---

## 1. OAuth 2.0 Overview

OAuth 2.0 คือ authorization framework ที่อนุญาตให้ application เข้าถึง resources ของ user โดยไม่ต้องรู้ password

```
OAuth 2.0 Roles:
┌─────────────────────────────────────────────────────┐
│  Resource Owner  → User ที่มี data                   │
│  Client          → Application ที่ต้องการ data        │
│  Authorization Server → ออก tokens                   │
│  Resource Server → API ที่มี data                    │
└─────────────────────────────────────────────────────┘

Authorization Code Flow:
User → Client → Auth Server → (code) → Client
Client → Auth Server → (access_token) → Client
Client → Resource Server → (data)
```

---

## 2. Authorization Code Flow

```go
// oauth/authorization_code.go
package oauth

import (
    "context"
    "crypto/rand"
    "encoding/base64"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "strings"
    "time"
)

type AuthCodeClient struct {
    ClientID     string
    ClientSecret string
    RedirectURI  string
    AuthURL      string
    TokenURL     string
    Scopes       []string
}

type TokenResponse struct {
    AccessToken  string `json:"access_token"`
    TokenType    string `json:"token_type"`
    ExpiresIn    int    `json:"expires_in"`
    RefreshToken string `json:"refresh_token"`
    IDToken      string `json:"id_token"` // OIDC
    Scope        string `json:"scope"`
}

// สร้าง state parameter (ป้องกัน CSRF)
func generateState() (string, error) {
    b := make([]byte, 32)
    _, err := rand.Read(b)
    if err != nil {
        return "", err
    }
    return base64.URLEncoding.EncodeToString(b), nil
}

// สร้าง Authorization URL
func (c *AuthCodeClient) GetAuthorizationURL(state string) string {
    params := url.Values{}
    params.Set("client_id", c.ClientID)
    params.Set("redirect_uri", c.RedirectURI)
    params.Set("response_type", "code")
    params.Set("state", state)
    params.Set("scope", strings.Join(c.Scopes, " "))
    
    return c.AuthURL + "?" + params.Encode()
}

// แลก authorization code เป็น token
func (c *AuthCodeClient) ExchangeCode(ctx context.Context, code string) (*TokenResponse, error) {
    data := url.Values{}
    data.Set("grant_type", "authorization_code")
    data.Set("code", code)
    data.Set("redirect_uri", c.RedirectURI)
    data.Set("client_id", c.ClientID)
    data.Set("client_secret", c.ClientSecret)
    
    req, err := http.NewRequestWithContext(ctx, "POST", c.TokenURL, strings.NewReader(data.Encode()))
    if err != nil {
        return nil, err
    }
    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
    
    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("token request failed: %w", err)
    }
    defer resp.Body.Close()
    
    body, _ := io.ReadAll(resp.Body)
    
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("token error: %s", string(body))
    }
    
    var tokenResp TokenResponse
    if err := json.Unmarshal(body, &tokenResp); err != nil {
        return nil, fmt.Errorf("failed to parse token response: %w", err)
    }
    
    return &tokenResp, nil
}

// Refresh token
func (c *AuthCodeClient) RefreshToken(ctx context.Context, refreshToken string) (*TokenResponse, error) {
    data := url.Values{}
    data.Set("grant_type", "refresh_token")
    data.Set("refresh_token", refreshToken)
    data.Set("client_id", c.ClientID)
    data.Set("client_secret", c.ClientSecret)
    
    req, _ := http.NewRequestWithContext(ctx, "POST", c.TokenURL, strings.NewReader(data.Encode()))
    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
    
    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var tokenResp TokenResponse
    json.NewDecoder(resp.Body).Decode(&tokenResp)
    
    return &tokenResp, nil
}

// HTTP Handler สำหรับ OAuth flow
type OAuthHandler struct {
    client    *AuthCodeClient
    stateStore map[string]time.Time // ควรใช้ Redis ใน production
    sessions  map[string]*TokenResponse
}

func (h *OAuthHandler) LoginHandler(w http.ResponseWriter, r *http.Request) {
    state, err := generateState()
    if err != nil {
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }
    
    // เก็บ state (expire ใน 10 นาที)
    h.stateStore[state] = time.Now().Add(10 * time.Minute)
    
    // Set state ใน cookie
    http.SetCookie(w, &http.Cookie{
        Name:     "oauth_state",
        Value:    state,
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteLaxMode,
        MaxAge:   600,
    })
    
    authURL := h.client.GetAuthorizationURL(state)
    http.Redirect(w, r, authURL, http.StatusFound)
}

func (h *OAuthHandler) CallbackHandler(w http.ResponseWriter, r *http.Request) {
    // ตรวจสอบ state
    stateCookie, err := r.Cookie("oauth_state")
    if err != nil {
        http.Error(w, "Missing state", http.StatusBadRequest)
        return
    }
    
    stateParam := r.URL.Query().Get("state")
    if stateCookie.Value != stateParam {
        http.Error(w, "State mismatch - CSRF detected", http.StatusBadRequest)
        return
    }
    
    // ตรวจสอบ error จาก Auth Server
    if errParam := r.URL.Query().Get("error"); errParam != "" {
        http.Error(w, "Auth error: "+errParam, http.StatusBadRequest)
        return
    }
    
    // แลก code เป็น token
    code := r.URL.Query().Get("code")
    token, err := h.client.ExchangeCode(r.Context(), code)
    if err != nil {
        http.Error(w, "Token exchange failed: "+err.Error(), http.StatusInternalServerError)
        return
    }
    
    // สร้าง session
    sessionID, _ := generateState()
    h.sessions[sessionID] = token
    
    http.SetCookie(w, &http.Cookie{
        Name:     "session_id",
        Value:    sessionID,
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteStrictMode,
        MaxAge:   3600,
    })
    
    http.Redirect(w, r, "/dashboard", http.StatusFound)
}
```

---

## 3. PKCE (Proof Key for Code Exchange)

PKCE ใช้กับ public clients (mobile apps, SPAs) ที่ไม่สามารถเก็บ client secret ได้อย่างปลอดภัย

```go
// oauth/pkce.go
package oauth

import (
    "crypto/rand"
    "crypto/sha256"
    "encoding/base64"
    "fmt"
    "net/url"
    "strings"
)

type PKCEVerifier struct {
    CodeVerifier  string
    CodeChallenge string
    Method        string
}

// สร้าง PKCE verifier และ challenge
func NewPKCEVerifier() (*PKCEVerifier, error) {
    // สร้าง random verifier (43-128 chars)
    b := make([]byte, 64)
    if _, err := rand.Read(b); err != nil {
        return nil, fmt.Errorf("failed to generate verifier: %w", err)
    }
    
    verifier := base64.RawURLEncoding.EncodeToString(b)
    
    // สร้าง challenge โดยใช้ SHA-256
    hash := sha256.Sum256([]byte(verifier))
    challenge := base64.RawURLEncoding.EncodeToString(hash[:])
    
    return &PKCEVerifier{
        CodeVerifier:  verifier,
        CodeChallenge: challenge,
        Method:        "S256",
    }, nil
}

// Authorization URL กับ PKCE
func (c *AuthCodeClient) GetPKCEAuthorizationURL(state string, pkce *PKCEVerifier) string {
    params := url.Values{}
    params.Set("client_id", c.ClientID)
    params.Set("redirect_uri", c.RedirectURI)
    params.Set("response_type", "code")
    params.Set("state", state)
    params.Set("scope", strings.Join(c.Scopes, " "))
    params.Set("code_challenge", pkce.CodeChallenge)
    params.Set("code_challenge_method", pkce.Method)
    
    return c.AuthURL + "?" + params.Encode()
}

// Exchange code กับ PKCE verifier
func (c *AuthCodeClient) ExchangeCodeWithPKCE(ctx context.Context, code string, verifier *PKCEVerifier) (*TokenResponse, error) {
    data := url.Values{}
    data.Set("grant_type", "authorization_code")
    data.Set("code", code)
    data.Set("redirect_uri", c.RedirectURI)
    data.Set("client_id", c.ClientID)
    data.Set("code_verifier", verifier.CodeVerifier)
    // ไม่ต้องส่ง client_secret สำหรับ public clients
    
    return doTokenRequest(ctx, c.TokenURL, data)
}
```

---

## 4. Client Credentials Flow

ใช้สำหรับ machine-to-machine communication (service-to-service)

```go
// oauth/client_credentials.go
package oauth

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "net/url"
    "strings"
    "sync"
    "time"
)

type ClientCredentialsClient struct {
    ClientID     string
    ClientSecret string
    TokenURL     string
    Scopes       []string
    
    mu          sync.RWMutex
    cachedToken *cachedToken
}

type cachedToken struct {
    token     *TokenResponse
    expiresAt time.Time
}

// Get token พร้อม auto-renewal
func (c *ClientCredentialsClient) GetToken(ctx context.Context) (string, error) {
    c.mu.RLock()
    cached := c.cachedToken
    c.mu.RUnlock()
    
    // ถ้า token ยังใช้ได้ (คืนก่อน expire 30 วินาที)
    if cached != nil && time.Now().Before(cached.expiresAt.Add(-30*time.Second)) {
        return cached.token.AccessToken, nil
    }
    
    c.mu.Lock()
    defer c.mu.Unlock()
    
    // Double-check หลัง lock
    if c.cachedToken != nil && time.Now().Before(c.cachedToken.expiresAt.Add(-30*time.Second)) {
        return c.cachedToken.token.AccessToken, nil
    }
    
    // ขอ token ใหม่
    token, err := c.fetchNewToken(ctx)
    if err != nil {
        return "", err
    }
    
    c.cachedToken = &cachedToken{
        token:     token,
        expiresAt: time.Now().Add(time.Duration(token.ExpiresIn) * time.Second),
    }
    
    return token.AccessToken, nil
}

func (c *ClientCredentialsClient) fetchNewToken(ctx context.Context) (*TokenResponse, error) {
    data := url.Values{}
    data.Set("grant_type", "client_credentials")
    data.Set("client_id", c.ClientID)
    data.Set("client_secret", c.ClientSecret)
    data.Set("scope", strings.Join(c.Scopes, " "))
    
    req, err := http.NewRequestWithContext(ctx, "POST", c.TokenURL, strings.NewReader(data.Encode()))
    if err != nil {
        return nil, err
    }
    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
    
    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("client credentials request failed: %w", err)
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("token request failed: %s", resp.Status)
    }
    
    var token TokenResponse
    if err := json.NewDecoder(resp.Body).Decode(&token); err != nil {
        return nil, err
    }
    
    return &token, nil
}

// HTTP Transport ที่ auto-inject token
type OAuthTransport struct {
    wrapped http.RoundTripper
    client  *ClientCredentialsClient
}

func NewOAuthTransport(client *ClientCredentialsClient) *OAuthTransport {
    return &OAuthTransport{
        wrapped: http.DefaultTransport,
        client:  client,
    }
}

func (t *OAuthTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    token, err := t.client.GetToken(req.Context())
    if err != nil {
        return nil, fmt.Errorf("failed to get token: %w", err)
    }
    
    // Clone request และใส่ token
    reqClone := req.Clone(req.Context())
    reqClone.Header.Set("Authorization", "Bearer "+token)
    
    return t.wrapped.RoundTrip(reqClone)
}

// ใช้งาน
func ExampleClientCredentials() {
    oauthClient := &ClientCredentialsClient{
        ClientID:     "my-service",
        ClientSecret: "super-secret",
        TokenURL:     "https://auth.example.com/oauth/token",
        Scopes:       []string{"read:users", "write:orders"},
    }
    
    httpClient := &http.Client{
        Transport: NewOAuthTransport(oauthClient),
    }
    
    // ทุก request จะมี token อัตโนมัติ
    resp, err := httpClient.Get("https://api.example.com/users")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    defer resp.Body.Close()
}
```

---

## 5. OpenID Connect (OIDC)

OIDC เพิ่ม authentication layer บน OAuth 2.0 โดยส่ง ID Token (JWT) ที่มีข้อมูล user

```go
// oidc/client.go
package oidc

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"
    
    "github.com/coreos/go-oidc/v3/oidc"
    "golang.org/x/oauth2"
)

type OIDCClient struct {
    provider    *oidc.Provider
    verifier    *oidc.IDTokenVerifier
    oauthConfig *oauth2.Config
}

func NewOIDCClient(ctx context.Context, issuerURL, clientID, clientSecret, redirectURI string) (*OIDCClient, error) {
    // Auto-discover OIDC configuration
    provider, err := oidc.NewProvider(ctx, issuerURL)
    if err != nil {
        return nil, fmt.Errorf("failed to create OIDC provider: %w", err)
    }
    
    // ID Token verifier
    verifier := provider.Verifier(&oidc.Config{
        ClientID: clientID,
    })
    
    // OAuth2 config
    oauthConfig := &oauth2.Config{
        ClientID:     clientID,
        ClientSecret: clientSecret,
        RedirectURL:  redirectURI,
        Endpoint:     provider.Endpoint(),
        Scopes: []string{
            oidc.ScopeOpenID,
            "profile",
            "email",
        },
    }
    
    return &OIDCClient{
        provider:    provider,
        verifier:    verifier,
        oauthConfig: oauthConfig,
    }, nil
}

type UserInfo struct {
    Sub           string `json:"sub"`
    Name          string `json:"name"`
    Email         string `json:"email"`
    EmailVerified bool   `json:"email_verified"`
    Picture       string `json:"picture"`
    GivenName     string `json:"given_name"`
    FamilyName    string `json:"family_name"`
}

func (c *OIDCClient) GetAuthURL(state, nonce string) string {
    return c.oauthConfig.AuthCodeURL(state,
        oidc.Nonce(nonce),
    )
}

func (c *OIDCClient) HandleCallback(ctx context.Context, code, nonce string) (*UserInfo, *oauth2.Token, error) {
    // Exchange code for token
    oauth2Token, err := c.oauthConfig.Exchange(ctx, code)
    if err != nil {
        return nil, nil, fmt.Errorf("code exchange failed: %w", err)
    }
    
    // Extract ID token
    rawIDToken, ok := oauth2Token.Extra("id_token").(string)
    if !ok {
        return nil, nil, fmt.Errorf("no id_token in response")
    }
    
    // Verify ID token
    idToken, err := c.verifier.Verify(ctx, rawIDToken)
    if err != nil {
        return nil, nil, fmt.Errorf("id_token verification failed: %w", err)
    }
    
    // ตรวจสอบ nonce
    var claims struct {
        Nonce string `json:"nonce"`
    }
    if err := idToken.Claims(&claims); err != nil {
        return nil, nil, err
    }
    if claims.Nonce != nonce {
        return nil, nil, fmt.Errorf("nonce mismatch")
    }
    
    // ดึง user info
    var userInfo UserInfo
    if err := idToken.Claims(&userInfo); err != nil {
        return nil, nil, err
    }
    
    return &userInfo, oauth2Token, nil
}

// JWT Validation middleware
func (c *OIDCClient) ValidateTokenMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := extractBearerToken(r)
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        idToken, err := c.verifier.Verify(r.Context(), token)
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }
        
        var claims UserInfo
        idToken.Claims(&claims)
        
        ctx := context.WithValue(r.Context(), "user", &claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

---

## 6. JWT Validation

```go
// jwt/validator.go
package jwt

import (
    "context"
    "crypto/rsa"
    "encoding/base64"
    "encoding/json"
    "fmt"
    "math/big"
    "net/http"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
)

type JWTValidator struct {
    issuer    string
    audience  string
    jwksURL   string
    keyCache  map[string]*rsa.PublicKey
    cacheTime time.Time
}

// JWKS (JSON Web Key Set) - ดึง public keys จาก Auth Server
type JWKSResponse struct {
    Keys []JWK `json:"keys"`
}

type JWK struct {
    Kty string `json:"kty"`
    Use string `json:"use"`
    N   string `json:"n"`
    E   string `json:"e"`
    Kid string `json:"kid"`
    Alg string `json:"alg"`
}

func (v *JWTValidator) getPublicKey(kid string) (*rsa.PublicKey, error) {
    // ตรวจสอบ cache (refresh ทุก 1 ชั่วโมง)
    if time.Since(v.cacheTime) > time.Hour || len(v.keyCache) == 0 {
        if err := v.refreshKeys(); err != nil {
            return nil, err
        }
    }
    
    key, ok := v.keyCache[kid]
    if !ok {
        return nil, fmt.Errorf("key %s not found", kid)
    }
    
    return key, nil
}

func (v *JWTValidator) refreshKeys() error {
    resp, err := http.Get(v.jwksURL)
    if err != nil {
        return fmt.Errorf("failed to fetch JWKS: %w", err)
    }
    defer resp.Body.Close()
    
    var jwks JWKSResponse
    if err := json.NewDecoder(resp.Body).Decode(&jwks); err != nil {
        return err
    }
    
    newCache := make(map[string]*rsa.PublicKey)
    for _, jwk := range jwks.Keys {
        if jwk.Use != "sig" || jwk.Kty != "RSA" {
            continue
        }
        
        key, err := parseRSAPublicKey(jwk)
        if err != nil {
            continue
        }
        
        newCache[jwk.Kid] = key
    }
    
    v.keyCache = newCache
    v.cacheTime = time.Now()
    
    return nil
}

func parseRSAPublicKey(jwk JWK) (*rsa.PublicKey, error) {
    nBytes, err := base64.RawURLEncoding.DecodeString(jwk.N)
    if err != nil {
        return nil, err
    }
    
    eBytes, err := base64.RawURLEncoding.DecodeString(jwk.E)
    if err != nil {
        return nil, err
    }
    
    n := new(big.Int).SetBytes(nBytes)
    
    var eInt int
    for _, b := range eBytes {
        eInt = eInt<<8 + int(b)
    }
    
    return &rsa.PublicKey{N: n, E: eInt}, nil
}

type Claims struct {
    jwt.RegisteredClaims
    UserID string   `json:"user_id"`
    Email  string   `json:"email"`
    Roles  []string `json:"roles"`
}

func (v *JWTValidator) ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodRSA); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        
        kid, ok := token.Header["kid"].(string)
        if !ok {
            return nil, fmt.Errorf("missing kid in token header")
        }
        
        return v.getPublicKey(kid)
    })
    
    if err != nil {
        return nil, fmt.Errorf("token parse failed: %w", err)
    }
    
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid claims")
    }
    
    // ตรวจสอบ issuer
    if claims.Issuer != v.issuer {
        return nil, fmt.Errorf("invalid issuer: %s", claims.Issuer)
    }
    
    // ตรวจสอบ audience
    if !claims.VerifyAudience(v.audience, true) {
        return nil, fmt.Errorf("invalid audience")
    }
    
    return claims, nil
}
```

---

## 7. Keycloak Integration

```go
// keycloak/client.go
package keycloak

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "net/url"
    "strings"
)

type KeycloakClient struct {
    baseURL      string
    realm        string
    clientID     string
    clientSecret string
    adminToken   string
}

func NewKeycloakClient(baseURL, realm, clientID, clientSecret string) *KeycloakClient {
    return &KeycloakClient{
        baseURL:      baseURL,
        realm:        realm,
        clientID:     clientID,
        clientSecret: clientSecret,
    }
}

// User Management ผ่าน Keycloak Admin API
type KeycloakUser struct {
    ID              string              `json:"id"`
    Username        string              `json:"username"`
    Email           string              `json:"email"`
    FirstName       string              `json:"firstName"`
    LastName        string              `json:"lastName"`
    Enabled         bool                `json:"enabled"`
    EmailVerified   bool                `json:"emailVerified"`
    Attributes      map[string][]string `json:"attributes,omitempty"`
    RealmRoles      []string            `json:"realmRoles,omitempty"`
}

func (kc *KeycloakClient) GetAdminToken(ctx context.Context, adminUser, adminPass string) error {
    data := url.Values{}
    data.Set("grant_type", "password")
    data.Set("client_id", "admin-cli")
    data.Set("username", adminUser)
    data.Set("password", adminPass)
    
    tokenURL := fmt.Sprintf("%s/realms/master/protocol/openid-connect/token", kc.baseURL)
    
    req, _ := http.NewRequestWithContext(ctx, "POST", tokenURL, strings.NewReader(data.Encode()))
    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    var tokenResp map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&tokenResp)
    
    token, ok := tokenResp["access_token"].(string)
    if !ok {
        return fmt.Errorf("failed to get admin token")
    }
    
    kc.adminToken = token
    return nil
}

func (kc *KeycloakClient) CreateUser(ctx context.Context, user KeycloakUser) error {
    userJSON, _ := json.Marshal(user)
    
    usersURL := fmt.Sprintf("%s/admin/realms/%s/users", kc.baseURL, kc.realm)
    
    req, _ := http.NewRequestWithContext(ctx, "POST", usersURL, strings.NewReader(string(userJSON)))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+kc.adminToken)
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusCreated {
        return fmt.Errorf("failed to create user: %s", resp.Status)
    }
    
    return nil
}

func (kc *KeycloakClient) AssignRole(ctx context.Context, userID, roleName string) error {
    // ดึง role ID ก่อน
    roleID, err := kc.getRoleID(ctx, roleName)
    if err != nil {
        return err
    }
    
    roles := []map[string]string{
        {"id": roleID, "name": roleName},
    }
    
    rolesJSON, _ := json.Marshal(roles)
    
    assignURL := fmt.Sprintf("%s/admin/realms/%s/users/%s/role-mappings/realm",
        kc.baseURL, kc.realm, userID)
    
    req, _ := http.NewRequestWithContext(ctx, "POST", assignURL, strings.NewReader(string(rolesJSON)))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+kc.adminToken)
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    return nil
}
```

---

## 8. Google OAuth

```go
// google/oauth.go
package google

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    
    "golang.org/x/oauth2"
    "golang.org/x/oauth2/google"
)

var googleOAuthConfig = &oauth2.Config{
    ClientID:     "YOUR_CLIENT_ID",
    ClientSecret: "YOUR_CLIENT_SECRET",
    RedirectURL:  "http://localhost:8080/auth/google/callback",
    Scopes: []string{
        "https://www.googleapis.com/auth/userinfo.email",
        "https://www.googleapis.com/auth/userinfo.profile",
    },
    Endpoint: google.Endpoint,
}

type GoogleUser struct {
    ID            string `json:"id"`
    Email         string `json:"email"`
    VerifiedEmail bool   `json:"verified_email"`
    Name          string `json:"name"`
    GivenName     string `json:"given_name"`
    FamilyName    string `json:"family_name"`
    Picture       string `json:"picture"`
}

func GetGoogleUser(ctx context.Context, accessToken string) (*GoogleUser, error) {
    userInfoURL := "https://www.googleapis.com/oauth2/v2/userinfo"
    
    req, _ := http.NewRequestWithContext(ctx, "GET", userInfoURL, nil)
    req.Header.Set("Authorization", "Bearer "+accessToken)
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("failed to get user info: %w", err)
    }
    defer resp.Body.Close()
    
    var user GoogleUser
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        return nil, err
    }
    
    return &user, nil
}

// Handler
func GoogleLoginHandler(w http.ResponseWriter, r *http.Request) {
    state, _ := generateState()
    
    http.SetCookie(w, &http.Cookie{
        Name:  "oauth_state",
        Value: state,
        Path:  "/",
    })
    
    url := googleOAuthConfig.AuthCodeURL(state, oauth2.AccessTypeOnline)
    http.Redirect(w, r, url, http.StatusTemporaryRedirect)
}

func GoogleCallbackHandler(w http.ResponseWriter, r *http.Request) {
    // ตรวจสอบ state
    state := r.URL.Query().Get("state")
    cookie, _ := r.Cookie("oauth_state")
    if state != cookie.Value {
        http.Error(w, "Invalid state", http.StatusBadRequest)
        return
    }
    
    code := r.URL.Query().Get("code")
    token, err := googleOAuthConfig.Exchange(r.Context(), code)
    if err != nil {
        http.Error(w, "Code exchange failed", http.StatusInternalServerError)
        return
    }
    
    user, err := GetGoogleUser(r.Context(), token.AccessToken)
    if err != nil {
        http.Error(w, "Failed to get user", http.StatusInternalServerError)
        return
    }
    
    // บันทึก user หรือ login
    fmt.Fprintf(w, "Welcome, %s! (%s)", user.Name, user.Email)
}
```

---

## 9. Custom OAuth Server

```go
// authserver/server.go
package authserver

import (
    "crypto/rand"
    "crypto/rsa"
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
)

type AuthServer struct {
    clients    map[string]*Client
    users      map[string]*User
    authCodes  map[string]*AuthCode
    privateKey *rsa.PrivateKey
    mu         sync.RWMutex
}

type Client struct {
    ID          string
    Secret      string
    RedirectURIs []string
    Scopes      []string
}

type User struct {
    ID       string
    Username string
    Password string // ควร hash
    Email    string
}

type AuthCode struct {
    Code        string
    ClientID    string
    UserID      string
    RedirectURI string
    Scopes      []string
    ExpiresAt   time.Time
    Used        bool
}

func NewAuthServer() (*AuthServer, error) {
    privateKey, err := rsa.GenerateKey(rand.Reader, 2048)
    if err != nil {
        return nil, err
    }
    
    return &AuthServer{
        clients:   make(map[string]*Client),
        users:     make(map[string]*User),
        authCodes: make(map[string]*AuthCode),
        privateKey: privateKey,
    }, nil
}

// Authorization endpoint
func (s *AuthServer) AuthorizeHandler(w http.ResponseWriter, r *http.Request) {
    clientID := r.URL.Query().Get("client_id")
    redirectURI := r.URL.Query().Get("redirect_uri")
    responseType := r.URL.Query().Get("response_type")
    scope := r.URL.Query().Get("scope")
    state := r.URL.Query().Get("state")
    
    // ตรวจสอบ client
    client, ok := s.clients[clientID]
    if !ok {
        http.Error(w, "Invalid client", http.StatusBadRequest)
        return
    }
    
    // ตรวจสอบ redirect URI
    validURI := false
    for _, uri := range client.RedirectURIs {
        if uri == redirectURI {
            validURI = true
            break
        }
    }
    if !validURI {
        http.Error(w, "Invalid redirect URI", http.StatusBadRequest)
        return
    }
    
    if responseType != "code" {
        http.Redirect(w, r, redirectURI+"?error=unsupported_response_type&state="+state, http.StatusFound)
        return
    }
    
    if r.Method == http.MethodGet {
        // แสดง login form
        fmt.Fprintf(w, `<html><body>
<form method="POST">
<input name="username" placeholder="Username">
<input name="password" type="password" placeholder="Password">
<input type="hidden" name="client_id" value="%s">
<input type="hidden" name="redirect_uri" value="%s">
<input type="hidden" name="state" value="%s">
<input type="hidden" name="scope" value="%s">
<button type="submit">Login</button>
</form></body></html>`, clientID, redirectURI, state, scope)
        return
    }
    
    // Process login
    username := r.FormValue("username")
    password := r.FormValue("password")
    
    user := s.authenticateUser(username, password)
    if user == nil {
        http.Error(w, "Invalid credentials", http.StatusUnauthorized)
        return
    }
    
    // สร้าง authorization code
    code, _ := generateRandomString(32)
    s.mu.Lock()
    s.authCodes[code] = &AuthCode{
        Code:        code,
        ClientID:    clientID,
        UserID:      user.ID,
        RedirectURI: redirectURI,
        Scopes:      strings.Split(scope, " "),
        ExpiresAt:   time.Now().Add(10 * time.Minute),
    }
    s.mu.Unlock()
    
    redirectURL := redirectURI + "?code=" + code + "&state=" + state
    http.Redirect(w, r, redirectURL, http.StatusFound)
}

// Token endpoint
func (s *AuthServer) TokenHandler(w http.ResponseWriter, r *http.Request) {
    grantType := r.FormValue("grant_type")
    
    switch grantType {
    case "authorization_code":
        s.handleAuthCodeGrant(w, r)
    case "client_credentials":
        s.handleClientCredentials(w, r)
    case "refresh_token":
        s.handleRefreshToken(w, r)
    default:
        http.Error(w, `{"error":"unsupported_grant_type"}`, http.StatusBadRequest)
    }
}

func (s *AuthServer) handleAuthCodeGrant(w http.ResponseWriter, r *http.Request) {
    code := r.FormValue("code")
    clientID := r.FormValue("client_id")
    clientSecret := r.FormValue("client_secret")
    
    // ตรวจสอบ client
    client, ok := s.clients[clientID]
    if !ok || client.Secret != clientSecret {
        w.WriteHeader(http.StatusUnauthorized)
        json.NewEncoder(w).Encode(map[string]string{"error": "invalid_client"})
        return
    }
    
    s.mu.Lock()
    authCode, ok := s.authCodes[code]
    if ok {
        authCode.Used = true
        delete(s.authCodes, code)
    }
    s.mu.Unlock()
    
    if !ok {
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]string{"error": "invalid_grant"})
        return
    }
    
    if time.Now().After(authCode.ExpiresAt) {
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]string{"error": "expired_grant"})
        return
    }
    
    // สร้าง JWT access token
    accessToken, err := s.createAccessToken(authCode.UserID, authCode.ClientID, authCode.Scopes)
    if err != nil {
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(TokenResponse{
        AccessToken: accessToken,
        TokenType:   "Bearer",
        ExpiresIn:   3600,
    })
}

func (s *AuthServer) createAccessToken(userID, clientID string, scopes []string) (string, error) {
    claims := jwt.MapClaims{
        "sub":    userID,
        "aud":    clientID,
        "iss":    "https://auth.example.com",
        "exp":    time.Now().Add(time.Hour).Unix(),
        "iat":    time.Now().Unix(),
        "scopes": scopes,
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodRS256, claims)
    token.Header["kid"] = "key-1"
    
    return token.SignedString(s.privateKey)
}

// JWKS endpoint
func (s *AuthServer) JWKSHandler(w http.ResponseWriter, r *http.Request) {
    pubKey := s.privateKey.Public().(*rsa.PublicKey)
    
    n := base64.RawURLEncoding.EncodeToString(pubKey.N.Bytes())
    e := base64.RawURLEncoding.EncodeToString(big.NewInt(int64(pubKey.E)).Bytes())
    
    jwks := map[string]interface{}{
        "keys": []interface{}{
            map[string]interface{}{
                "kty": "RSA",
                "use": "sig",
                "alg": "RS256",
                "kid": "key-1",
                "n":   n,
                "e":   e,
            },
        },
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(jwks)
}
```

---

## Workshop: SSO System

```go
// workshop/sso/main.go
package main

import (
    "context"
    "encoding/json"
    "log"
    "net/http"
    "time"
    
    "github.com/coreos/go-oidc/v3/oidc"
    "golang.org/x/oauth2"
)

type SSOServer struct {
    oidcClient  *OIDCClient
    sessions    *SessionStore
    users       UserRepository
}

type Session struct {
    UserID       string
    AccessToken  string
    RefreshToken string
    IDToken      string
    ExpiresAt    time.Time
}

type SessionStore struct {
    sessions map[string]*Session
}

func NewSSOServer(issuerURL, clientID, clientSecret, redirectURI string) (*SSOServer, error) {
    ctx := context.Background()
    
    oidcClient, err := NewOIDCClient(ctx, issuerURL, clientID, clientSecret, redirectURI)
    if err != nil {
        return nil, err
    }
    
    return &SSOServer{
        oidcClient: oidcClient,
        sessions:   &SessionStore{sessions: make(map[string]*Session)},
    }, nil
}

func (s *SSOServer) Routes() http.Handler {
    mux := http.NewServeMux()
    
    mux.HandleFunc("/login", s.loginHandler)
    mux.HandleFunc("/callback", s.callbackHandler)
    mux.HandleFunc("/logout", s.logoutHandler)
    mux.HandleFunc("/me", s.requireAuth(s.meHandler))
    mux.HandleFunc("/refresh", s.refreshHandler)
    
    return mux
}

func (s *SSOServer) loginHandler(w http.ResponseWriter, r *http.Request) {
    state, _ := generateState()
    nonce, _ := generateState()
    
    // เก็บ state และ nonce ใน cookie
    http.SetCookie(w, &http.Cookie{Name: "oauth_state", Value: state, HttpOnly: true, Secure: true})
    http.SetCookie(w, &http.Cookie{Name: "oauth_nonce", Value: nonce, HttpOnly: true, Secure: true})
    
    // เพิ่ม return URL
    if returnURL := r.URL.Query().Get("return_url"); returnURL != "" {
        http.SetCookie(w, &http.Cookie{Name: "return_url", Value: returnURL})
    }
    
    authURL := s.oidcClient.GetAuthURL(state, nonce)
    http.Redirect(w, r, authURL, http.StatusFound)
}

func (s *SSOServer) callbackHandler(w http.ResponseWriter, r *http.Request) {
    stateCookie, _ := r.Cookie("oauth_state")
    nonceCookie, _ := r.Cookie("oauth_nonce")
    
    if r.URL.Query().Get("state") != stateCookie.Value {
        http.Error(w, "Invalid state", http.StatusBadRequest)
        return
    }
    
    userInfo, oauth2Token, err := s.oidcClient.HandleCallback(
        r.Context(),
        r.URL.Query().Get("code"),
        nonceCookie.Value,
    )
    if err != nil {
        http.Error(w, "Auth failed: "+err.Error(), http.StatusInternalServerError)
        return
    }
    
    // สร้าง session
    sessionID, _ := generateState()
    expiresAt := time.Now().Add(24 * time.Hour)
    
    s.sessions.sessions[sessionID] = &Session{
        UserID:       userInfo.Sub,
        AccessToken:  oauth2Token.AccessToken,
        RefreshToken: oauth2Token.RefreshToken,
        ExpiresAt:    expiresAt,
    }
    
    http.SetCookie(w, &http.Cookie{
        Name:     "session_id",
        Value:    sessionID,
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteStrictMode,
        Expires:  expiresAt,
    })
    
    // Redirect ไป return URL หรือ dashboard
    returnURL := "/"
    if c, err := r.Cookie("return_url"); err == nil {
        returnURL = c.Value
    }
    
    http.Redirect(w, r, returnURL, http.StatusFound)
}

func (s *SSOServer) requireAuth(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        sessionCookie, err := r.Cookie("session_id")
        if err != nil {
            http.Redirect(w, r, "/login?return_url="+r.URL.Path, http.StatusFound)
            return
        }
        
        session, ok := s.sessions.sessions[sessionCookie.Value]
        if !ok || time.Now().After(session.ExpiresAt) {
            http.Redirect(w, r, "/login", http.StatusFound)
            return
        }
        
        ctx := context.WithValue(r.Context(), "session", session)
        next(w, r.WithContext(ctx))
    }
}

func (s *SSOServer) meHandler(w http.ResponseWriter, r *http.Request) {
    session := r.Context().Value("session").(*Session)
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{
        "user_id": session.UserID,
    })
}

func main() {
    sso, err := NewSSOServer(
        "https://accounts.google.com",
        "YOUR_GOOGLE_CLIENT_ID",
        "YOUR_GOOGLE_CLIENT_SECRET",
        "http://localhost:8080/callback",
    )
    if err != nil {
        log.Fatal(err)
    }
    
    log.Println("SSO server running on :8080")
    log.Fatal(http.ListenAndServe(":8080", sso.Routes()))
}
```

---

## สรุป

| Flow | ใช้เมื่อ | Security |
|------|---------|---------|
| Auth Code | Web app มี server | สูง |
| Auth Code + PKCE | SPA, Mobile | สูงมาก |
| Client Credentials | Service-to-service | สูง |
| Device Code | TV, IoT | สูง |
| Implicit (deprecated) | - | ต่ำ (อย่าใช้) |

### Best Practices
1. ใช้ PKCE เสมอสำหรับ public clients
2. Validate ทุก token field (iss, aud, exp, nonce)
3. Rotate refresh tokens
4. ใช้ short-lived access tokens (1 ชั่วโมง)
5. Store tokens ใน httpOnly cookies
6. Implement CSRF protection ด้วย state parameter
7. ใช้ HTTPS เสมอ

---

*จบ Part 70: OAuth 2.0 และ OpenID Connect*
