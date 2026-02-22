# Security & Authentication - Interview Q&A

> 12+ questions covering JWT, OAuth, CORS, OWASP, and Web3 auth

---

## Table of Contents

- [Authentication](#authentication)
- [Authorization](#authorization)
- [Web Security (OWASP)](#web-security-owasp)
- [Web3 Authentication](#web3-authentication)

---

## Authentication

### Q1: JWT authentication with refresh tokens.

**Answer:**

```typescript
@Injectable()
export class AuthService {
  constructor(
    private jwtService: JwtService,
    private userRepo: UserRepository,
  ) {}

  async login(email: string, password: string) {
    const user = await this.userRepo.findByEmail(email);
    if (!user || !await bcrypt.compare(password, user.password)) {
      throw new UnauthorizedException('Invalid credentials');
    }

    return this.generateTokens(user);
  }

  private generateTokens(user: User) {
    // Access token: short-lived, contains user info
    const accessToken = this.jwtService.sign(
      { sub: user.id, email: user.email, role: user.role },
      { secret: process.env.JWT_SECRET, expiresIn: '15m' }
    );

    // Refresh token: long-lived, minimal claims
    const refreshToken = this.jwtService.sign(
      { sub: user.id, version: user.tokenVersion },
      { secret: process.env.JWT_REFRESH_SECRET, expiresIn: '7d' }
    );

    return { accessToken, refreshToken };
  }

  async refresh(refreshToken: string) {
    try {
      const payload = this.jwtService.verify(refreshToken, {
        secret: process.env.JWT_REFRESH_SECRET,
      });

      const user = await this.userRepo.findById(payload.sub);

      // Token version check (for revocation)
      if (!user || user.tokenVersion !== payload.version) {
        throw new UnauthorizedException('Token revoked');
      }

      // Token rotation: issue new pair, old refresh token is single-use
      return this.generateTokens(user);
    } catch {
      throw new UnauthorizedException('Invalid refresh token');
    }
  }

  async logout(userId: string) {
    // Increment version → all existing refresh tokens become invalid
    await this.userRepo.incrementTokenVersion(userId);
  }
}

// Token storage:
// Access Token  → Memory (JS variable) or short-lived httpOnly cookie
// Refresh Token → httpOnly, Secure, SameSite=Strict cookie
// NEVER localStorage → XSS vulnerable

// Cookie setup:
res.cookie('refreshToken', refreshToken, {
  httpOnly: true,    // not accessible via JavaScript
  secure: true,      // HTTPS only
  sameSite: 'strict', // CSRF protection
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
  path: '/api/auth/refresh', // only sent to refresh endpoint
});
```

**JWT structure:**
```
Header.Payload.Signature

Header:   { "alg": "HS256", "typ": "JWT" }
Payload:  { "sub": "user123", "email": "awais@...", "iat": 1234, "exp": 1234 }
Signature: HMACSHA256(base64(header) + "." + base64(payload), secret)
```

---

### Q2: OAuth 2.0 / OpenID Connect.

**Answer:**

```
Authorization Code Flow (most secure):
═══════════════════════════════════════

1. User clicks "Login with Google"
2. App redirects to Google:
   GET https://accounts.google.com/o/oauth2/v2/auth?
     client_id=YOUR_ID&
     redirect_uri=https://myapp.com/callback&
     response_type=code&
     scope=openid email profile&
     state=random_csrf_token

3. User logs in at Google, consents to permissions
4. Google redirects back with auth code:
   GET https://myapp.com/callback?code=AUTH_CODE&state=random_csrf_token

5. Server exchanges code for tokens (server-to-server, code is single-use):
   POST https://oauth2.googleapis.com/token
   { code, client_id, client_secret, redirect_uri, grant_type: 'authorization_code' }

6. Response: { access_token, refresh_token, id_token, expires_in }

7. Use access_token to call Google APIs (or create local session)
```

```typescript
// NestJS with Passport.js
@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy, 'google') {
  constructor(private authService: AuthService) {
    super({
      clientID: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
      callbackURL: '/api/auth/google/callback',
      scope: ['email', 'profile'],
    });
  }

  async validate(accessToken: string, refreshToken: string, profile: Profile) {
    const user = await this.authService.findOrCreateFromOAuth({
      email: profile.emails[0].value,
      name: profile.displayName,
      provider: 'google',
      providerId: profile.id,
    });
    return user;
  }
}

@Controller('auth')
export class AuthController {
  @Get('google')
  @UseGuards(AuthGuard('google'))
  googleLogin() {} // redirects to Google

  @Get('google/callback')
  @UseGuards(AuthGuard('google'))
  async googleCallback(@Req() req, @Res() res) {
    const tokens = await this.authService.generateTokens(req.user);
    res.redirect(`${FRONTEND_URL}/auth/callback?token=${tokens.accessToken}`);
  }
}

// OAuth flows comparison:
// Authorization Code:  Server apps (most secure, code exchanged server-side)
// Auth Code + PKCE:    SPAs/Mobile (no client_secret needed)
// Client Credentials:  Machine-to-machine (no user involved)
// Implicit:            DEPRECATED (token in URL, insecure)
```

---

## Authorization

### Q3: RBAC (Role-Based Access Control) in NestJS.

**Answer:**

```typescript
// 1. Roles decorator
export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);

// 2. Roles guard
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) return true; // no roles required

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some(role => user.role === role);
  }
}

// 3. Usage
@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
export class AdminController {
  @Get('users')
  @Roles('admin')
  getUsers() { /* admin only */ }

  @Get('trades')
  @Roles('admin', 'trader')
  getTrades() { /* admin or trader */ }
}

// 4. CASL (Attribute-Based Access Control) for fine-grained permissions
import { AbilityBuilder, PureAbility } from '@casl/ability';

type Actions = 'create' | 'read' | 'update' | 'delete' | 'manage';
type Subjects = 'Trade' | 'User' | 'all';

export function defineAbility(user: User) {
  const { can, cannot, build } = new AbilityBuilder<PureAbility<[Actions, Subjects]>>(PureAbility);

  if (user.role === 'admin') {
    can('manage', 'all'); // admin can do everything
  } else if (user.role === 'trader') {
    can('create', 'Trade');
    can('read', 'Trade', { userId: user.id });    // own trades only
    can('update', 'Trade', { userId: user.id, status: 'pending' }); // own pending only
    cannot('delete', 'Trade');
  } else {
    can('read', 'Trade', { userId: user.id });     // read own trades
  }

  return build();
}
```

---

## Web Security (OWASP)

### Q4: OWASP Top 10 - How to prevent common vulnerabilities.

**Answer:**

```typescript
// 1. INJECTION (SQL, NoSQL, Command)
// ❌ Bad
const query = `SELECT * FROM users WHERE email = '${email}'`;
// ✅ Good - parameterized queries
const result = await db.query('SELECT * FROM users WHERE email = $1', [email]);
// ✅ Good - ORM (TypeORM, Prisma)
const user = await userRepo.findOne({ where: { email } });

// 2. XSS (Cross-Site Scripting)
// ❌ Bad - React dangerouslySetInnerHTML
<div dangerouslySetInnerHTML={{ __html: userInput }} />
// ✅ Good - React auto-escapes by default
<div>{userInput}</div>
// ✅ Good - sanitize if HTML is needed
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(html) }} />
// ✅ CSP headers
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    },
  },
}));

// 3. CSRF (Cross-Site Request Forgery)
// ✅ SameSite cookies
res.cookie('session', token, { sameSite: 'strict' });
// ✅ CSRF tokens for forms
// ✅ Check Origin/Referer headers
// ✅ Use POST for mutations (not GET)

// 4. BROKEN AUTHENTICATION
// ✅ bcrypt for passwords (10+ rounds)
const hash = await bcrypt.hash(password, 12);
// ✅ Rate limit login attempts
// ✅ MFA for sensitive operations
// ✅ Account lockout after N failures
// ✅ Strong password requirements

// 5. SENSITIVE DATA EXPOSURE
// ✅ HTTPS everywhere (HSTS header)
// ✅ Encrypt data at rest (AES-256)
// ✅ Don't log sensitive data (passwords, tokens, PII)
// ✅ Use environment variables for secrets
// ✅ Mask sensitive fields in API responses
function sanitizeUser(user: User) {
  const { password, tokenVersion, ...safe } = user;
  return safe;
}

// 6. MASS ASSIGNMENT
// ❌ Bad
const user = await userRepo.save(req.body); // attacker can set role: 'admin'
// ✅ Good - whitelist allowed fields
class CreateUserDto {
  @IsEmail()
  email: string;
  @IsString()
  name: string;
  // role is NOT here → can't be set by user
}
// ✅ class-validator + whitelist: true
app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
```

---

### Q5: Explain CORS (Cross-Origin Resource Sharing).

**Answer:**

```
CORS prevents web pages from making requests to a different origin.

Origin = protocol + domain + port
https://myapp.com:443 ≠ https://api.myapp.com:443 (different subdomain)
https://myapp.com:443 ≠ http://myapp.com:80 (different protocol)

How CORS works:
1. Browser sends OPTIONS preflight request (for non-simple requests)
2. Server responds with allowed origins, methods, headers
3. If allowed, browser sends actual request
4. If not allowed, browser blocks the response

Preflight triggers:
- Methods other than GET, POST, HEAD
- Custom headers
- Content-Type other than form-data, urlencoded, text/plain
```

```typescript
// NestJS CORS configuration
app.enableCors({
  origin: [
    'https://myapp.com',
    'https://admin.myapp.com',
    process.env.NODE_ENV === 'development' && 'http://localhost:3000',
  ].filter(Boolean),
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,       // allow cookies
  maxAge: 86400,           // cache preflight for 24h
});

// Common CORS errors and fixes:
// "No 'Access-Control-Allow-Origin' header"
//   → Server needs to return the header
// "Credentials flag is true but Access-Control-Allow-Origin is '*'"
//   → Can't use wildcard with credentials; specify exact origins
// "Request header field X not allowed"
//   → Add the header to allowedHeaders
```

---

## Web3 Authentication

### Q6: Wallet-based authentication (Sign-In With Ethereum / Solana).

**Answer:**

```typescript
// Wallet auth flow:
// 1. Server sends random nonce
// 2. User signs the nonce with their wallet
// 3. Server verifies the signature
// 4. If valid → create session/JWT

// Backend (NestJS)
@Controller('auth')
export class Web3AuthController {
  @Post('nonce')
  async getNonce(@Body('walletAddress') address: string) {
    const nonce = randomBytes(32).toString('hex');
    await this.redis.setex(`nonce:${address}`, 300, nonce); // 5 min expiry
    return { nonce, message: `Sign this to login: ${nonce}` };
  }

  @Post('verify')
  async verify(@Body() dto: VerifyDto) {
    const { walletAddress, signature, message } = dto;

    // Get stored nonce
    const storedNonce = await this.redis.get(`nonce:${walletAddress}`);
    if (!storedNonce || !message.includes(storedNonce)) {
      throw new UnauthorizedException('Invalid or expired nonce');
    }

    // Verify signature (Solana)
    const isValid = nacl.sign.detached.verify(
      new TextEncoder().encode(message),
      bs58.decode(signature),
      new PublicKey(walletAddress).toBytes(),
    );

    if (!isValid) throw new UnauthorizedException('Invalid signature');

    // Delete nonce (prevent replay)
    await this.redis.del(`nonce:${walletAddress}`);

    // Find or create user
    const user = await this.authService.findOrCreateByWallet(walletAddress);
    return this.authService.generateTokens(user);
  }
}

// Frontend (with Privy - handles all this automatically)
const { login, authenticated, user } = usePrivy();

// Or with wallet adapter directly:
const { publicKey, signMessage } = useWallet();

async function loginWithWallet() {
  const { nonce, message } = await api.getNonce(publicKey.toString());
  const signature = await signMessage(new TextEncoder().encode(message));
  const { accessToken } = await api.verify({
    walletAddress: publicKey.toString(),
    signature: bs58.encode(signature),
    message,
  });
  setToken(accessToken);
}
```

**Why wallet auth is secure:**
- Signature proves ownership of private key
- Nonce prevents replay attacks
- No password to steal
- Decentralized identity (user controls their keys)

---

## References & Deep Dive Resources

### Authentication
| Topic | Resource |
|---|---|
| JWT.io | [jwt.io](https://jwt.io/) — JWT decoder & debugger |
| JWT Handbook (Auth0) | [auth0.com/resources/ebooks/jwt-handbook](https://auth0.com/resources/ebooks/jwt-handbook) |
| Access + Refresh Tokens | [Auth0 - Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation) |
| Bcrypt | [npmjs.com/package/bcrypt](https://www.npmjs.com/package/bcrypt) |
| Passport.js Docs | [passportjs.org](https://www.passportjs.org/) |
| NestJS Authentication | [NestJS - Authentication](https://docs.nestjs.com/security/authentication) |

### OAuth 2.0 / OIDC
| Topic | Resource |
|---|---|
| OAuth 2.0 Simplified | [oauth.net/2/](https://oauth.net/2/) |
| OAuth 2.0 Flows Explained | [Auth0 - Which OAuth 2.0 Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/which-oauth-2-0-flow-should-i-use) |
| OpenID Connect | [openid.net/developers/how-connect-works](https://openid.net/developers/how-connect-works/) |
| PKCE Explained | [Auth0 - PKCE](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce) |
| OAuth Playground | [developers.google.com/oauthplayground](https://developers.google.com/oauthplayground/) |

### Authorization (RBAC / ABAC)
| Topic | Resource |
|---|---|
| CASL.js | [casl.js.org](https://casl.js.org/) — Isomorphic authorization library |
| NestJS Authorization | [NestJS - Authorization](https://docs.nestjs.com/security/authorization) |
| RBAC vs ABAC | [Auth0 - RBAC vs ABAC](https://auth0.com/blog/role-based-access-control-rbac-and-react-apps/) |

### Web Security (OWASP)
| Topic | Resource |
|---|---|
| OWASP Top 10 | [owasp.org/Top10](https://owasp.org/www-project-top-ten/) |
| OWASP Cheat Sheets | [cheatsheetseries.owasp.org](https://cheatsheetseries.owasp.org/) — Security cheat sheets |
| XSS Prevention | [OWASP - XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) |
| SQL Injection | [OWASP - SQL Injection](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html) |
| CSRF Prevention | [OWASP - CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) |
| CORS Explained | [MDN - CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) |
| Content Security Policy | [MDN - CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) |
| Helmet.js | [helmetjs.github.io](https://helmetjs.github.io/) — Security headers for Express |
| DOMPurify | [github.com/cure53/DOMPurify](https://github.com/cure53/DOMPurify) — XSS sanitizer |

### Web3 Auth
| Topic | Resource |
|---|---|
| Sign-In With Ethereum (SIWE) | [login.xyz](https://login.xyz/) — SIWE standard |
| Privy Docs | [docs.privy.io](https://docs.privy.io/) |
| Wallet-based Auth | [Phantom - Signing Messages](https://docs.phantom.app/solana/signing-a-message) |
| NaCl (TweetNaCl.js) | [tweetnacl.js.org](https://tweetnacl.js.org/) — Crypto library for signature verification |

### Security Testing
| Topic | Resource |
|---|---|
| OWASP ZAP | [zaproxy.org](https://www.zaproxy.org/) — Free security scanner |
| Snyk | [snyk.io](https://snyk.io/) — Dependency vulnerability scanning |
| npm audit | [docs.npmjs.com/cli/audit](https://docs.npmjs.com/cli/v10/commands/npm-audit) |

---

> **Back to main**: [INTERVIEW_ROADMAP.md](../INTERVIEW_ROADMAP.md)
> **Prev**: [AI Integration](./09-ai-integration.md) | **Next**: [Behavioral & DSA](./11-behavioral-dsa.md)
