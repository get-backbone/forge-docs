---
title: "0016. Notification Service Rate Limiting Strategy"
summary: "The Notification Service sends notifications to external providers (AWS SES, Twilio, etc.), each with different API rate limits and constraints:"
---

**Status:** Accepted  
**Date:** 2026-01-23  
**Context:** Outbound rate limiting for SES, Twilio, and future providers so limits are respected and critical messages are not starved.

## **Context**

The Notification Service sends notifications to external providers (AWS SES, Twilio, etc.), each with different API rate limits and constraints:

- **AWS SES:** 1 email/sec (sandbox), 14 emails/sec (production), burst limits
- **Twilio SMS:** Account-specific limits (varies by account tier)
- **Future Providers:** Each will have their own constraints

The existing `libs/security` rate limiting framework (`RateLimitRequestFilter`) is designed for **incoming HTTP request** rate limiting (per-user, per-service, per-IP). This is fundamentally different from what the notification service needs: **outgoing notification rate limiting** to external providers with provider-specific constraints.

**Requirements:**
- Respect provider-specific API limits (SES, Twilio, etc.)
- Ensure critical notifications (password resets) are not blocked
- Support priority-based throttling
- Handle provider rate limit errors gracefully

---

## **Decision**

The Notification Service will implement **per-provider rate limiting** with **priority-aware throttling**.

### **Rate Limiting Strategy:**

1. **Per-Provider Rate Limiting**
   - Each provider (AWS SES, Twilio, etc.) has its own rate limiter
   - Provider-specific limits configured per provider
   - Limits respect external service constraints

2. **Priority-Aware Throttling**
   - CRITICAL notifications: Bypass or have very high limits (e.g., 1000/min)
   - HIGH notifications: High limits (e.g., 500/min)
   - NORMAL notifications: Standard limits (e.g., 100/min)
   - LOW notifications: Lower limits (e.g., 50/min), may be throttled further under load

3. **Capacity Management**
   - When system load > 80%: Throttle LOW priority notifications
   - When system load > 95%: Pause LOW priority processing
   - CRITICAL and HIGH always processed regardless of load

### **Implementation:**

- Custom rate limiter in `infrastructure/provider/` or `infrastructure/retry/`
- Provider-specific limits configured per provider
- Priority-aware: CRITICAL/HIGH may bypass or have higher limits, NORMAL/LOW respect provider limits
- Rate limit errors trigger retry logic with exponential backoff

---

## **Rationale**

### **Why Per-Provider (Not Per-Channel or Per-Priority Only):**

1. **Provider Constraints** - Each provider has different limits (SES ≠ Twilio)
2. **Granularity** - More granular than per-channel (multiple providers per channel)
3. **Accuracy** - Respects actual external service constraints
4. **Flexibility** - Can add new providers without affecting existing ones

### **Why Not Use Existing `libs/security` Rate Limiting:**

1. **Different Purpose** - Existing framework is for incoming HTTP requests, not outgoing provider calls
2. **Provider-Specific** - Need provider-specific limits, not user/service-based
3. **Priority-Aware** - Need priority-based throttling, not just flat limits
4. **Custom Logic** - Requires custom logic for provider failover and retry

### **Why Priority-Aware:**

1. **Critical Notifications** - Password resets, account lockouts must not be blocked
2. **Graceful Degradation** - Low-priority notifications can be delayed during high load
3. **Resource Optimization** - Process important notifications first
4. **User Experience** - Critical notifications always get through

---

## **Consequences**

### **Positive:**

- **Provider Compliance** - Respects external service rate limits
- **Critical Notifications Protected** - CRITICAL/HIGH notifications always processed
- **Graceful Degradation** - System degrades gracefully under load
- **Flexibility** - Easy to add new providers with their own limits
- **Cost Optimization** - Prevents rate limit errors and associated costs

### **Negative / Tradeoffs:**

- **Custom Implementation** - Requires custom rate limiter (not using existing framework)
- **Configuration Overhead** - Must configure limits per provider
- **Complexity** - Priority-aware logic adds complexity
- **Monitoring** - Need metrics for rate limiting per provider/priority

### **Mitigations:**

- **Reuse Patterns** - Can reuse rate limiting patterns from `libs/security` (Bucket4j, etc.)
- **Configuration** - Provider limits in configuration files
- **Metrics** - Rate limiting metrics integrated with existing metrics infrastructure
- **Documentation** - Clear documentation of provider limits and configuration

---

## **Implementation**

### **Rate Limiter Structure:**

```java
@ApplicationScoped
public class ProviderRateLimiter {
    private final Map<String, RateLimiter> providerLimiters;
    
    public boolean tryConsume(String provider, NotificationPriority priority) {
        RateLimiter limiter = providerLimiters.get(provider);
        // Priority-aware: CRITICAL bypasses, LOW respects strict limits
        return limiter.tryConsume(priority);
    }
}
```

### **Configuration:**

```properties
# AWS SES Rate Limits
notification.rate-limit.ses.critical.per-minute=1000
notification.rate-limit.ses.high.per-minute=500
notification.rate-limit.ses.normal.per-minute=100
notification.rate-limit.ses.low.per-minute=50

# Twilio Rate Limits (when implemented)
notification.rate-limit.twilio.critical.per-minute=1000
notification.rate-limit.twilio.high.per-minute=500
notification.rate-limit.twilio.normal.per-minute=100
notification.rate-limit.twilio.low.per-minute=50
```

### **Integration:**

- Rate limiter called before provider.send()
- If rate limit exceeded, notification queued for retry
- Retry respects priority (CRITICAL retried immediately, LOW delayed)

---

## **Related Decisions**

- **ADR-0015:** Fire-and-Forget / Asynchronous Messaging Pattern (priority-based processing)
- **ADR-0011:** Stateless JWT Authentication (stateless service design)

---

## **Future Considerations**

**Distributed Rate Limiting:**
- When multiple service instances, may need distributed rate limiter (Redis-backed)
- Current: Per-instance rate limiting (acceptable for stateless design)

**Dynamic Rate Limit Detection:**
- Future: Detect provider rate limits dynamically (from error responses)
- Current: Static configuration

---
