# OIDC Implementation Comparison Analysis

**Date:** 2025-11-17
**Plugin:** discourse-logto-mobile-session
**Comparison:** UserProvisioner vs Official OIDC Plugin

---

## Executive Summary

Your `UserProvisioner` implementation has **3 critical bugs** and **several missing features** compared to the official OIDC implementation. While it works for basic scenarios, it will fail in edge cases involving:

1. **Users switching Logto accounts with same email**
2. **Existing users without UserAssociatedAccount records**
3. **Staged users converting to real accounts**

**Recommendation:** Apply the fixes below to align with official patterns.

---

## Critical Bugs

### 🔴 BUG #1: Missing Duplicate Association Cleanup

**Location:** `user_provisioner.rb:67`

**Problem:**
```ruby
# Current code (LINE 67-76):
UserAssociatedAccount.create!(
  provider_name: 'oidc',
  provider_uid: @user_info[:sub],
  user_id: user.id,
  extra: { ... }.to_json
)
```

**Database Constraint:**
```sql
-- From user_associated_account.rb:31
UNIQUE INDEX associated_accounts_provider_user (provider_name, user_id)
```

**This means:** A user can only have ONE 'oidc' association at a time.

**Failure Scenario:**
1. User creates Logto account A (`sub: "abc"`, email: `[email protected]`)
2. Plugin creates: `UserAssociatedAccount(provider_uid: "abc", user_id: 1)`
3. User deletes Logto account A, creates new account B (`sub: "xyz"`, SAME email)
4. Plugin finds user by email ✓
5. Plugin tries: `UserAssociatedAccount.create!(provider_uid: "xyz", user_id: 1)`
6. **💥 CRASH: ActiveRecord::RecordNotUnique** (user 1 already has 'oidc' association)

**Official OIDC Solution** (`managed_authenticator.rb:73`):
```ruby
if match_by_email && association.user.nil? && (user = find_user_by_email(auth_token))
  # Destroy existing associations BEFORE linking new one
  UserAssociatedAccount.where(user: user, provider_name: auth_token[:provider]).destroy_all
  association.user = user
end
```

**Required Fix:**
```ruby
# user_provisioner.rb:66 - ADD BEFORE creating association
UserAssociatedAccount.where(user: user, provider_name: 'oidc').destroy_all

UserAssociatedAccount.create!(...)
```

---

### 🔴 BUG #2: Missing Association for Existing Users

**Location:** `user_provisioner.rb:90-103` (update_user_info method)

**Problem:**
When finding an existing user, the code only updates custom fields but **doesn't ensure UserAssociatedAccount exists**.

**Failure Scenario:**
1. User exists in Discourse (created via email/password or web OIDC)
2. User logs in via mobile app for FIRST time
3. `find_existing_user` finds the user by email ✓
4. `update_user_info` only updates custom fields ✗
5. **UserAssociatedAccount is NEVER created**
6. User can't be linked to Logto account properly

**Current Code:**
```ruby
def update_user_info(user)
  # Only updates custom fields, no association handling!
  user.custom_fields['logto_sub'] = @user_info[:sub]
  user.save_custom_fields(true)
end
```

**Required Fix:**
```ruby
def update_user_info(user)
  # Update name if changed
  if @user_info[:name].present? && user.name != @user_info[:name]
    user.name = @user_info[:name]
    user.save!
  end

  # Update custom fields
  user.custom_fields['logto_sub'] = @user_info[:sub]
  user.custom_fields['logto_email_verified'] = @user_info[:email_verified]
  user.custom_fields['logto_last_auth'] = Time.now.iso8601
  user.save_custom_fields(true)

  # ⭐ ENSURE UserAssociatedAccount exists
  ensure_user_association(user)
end

private

def ensure_user_association(user)
  association = UserAssociatedAccount.find_or_initialize_by(
    provider_name: 'oidc',
    provider_uid: @user_info[:sub]
  )

  # If association exists but linked to different user, destroy old ones
  if association.user && association.user.id != user.id
    UserAssociatedAccount.where(user: user, provider_name: 'oidc').destroy_all
  end

  association.user = user
  association.info = {
    email: @user_info[:email],
    name: @user_info[:name],
    picture: @user_info[:picture]
  }
  association.extra = {
    email_verified: @user_info[:email_verified],
    created_via: 'mobile_session_exchange'
  }
  association.last_used = Time.zone.now
  association.save!
end
```

---

### 🔴 BUG #3: UserAssociatedAccount Missing Required Fields

**Location:** `user_provisioner.rb:67-76`

**Problem:**
Your code only populates `extra` field, but official OIDC uses `info`, `credentials`, and `last_used` fields.

**Schema** (`user_associated_account.rb:22-24`):
```ruby
#  info          :jsonb            not null
#  credentials   :jsonb            not null
#  extra         :jsonb            not null
#  last_used     :datetime         not null
```

**Official Pattern** (`managed_authenticator.rb:85-89`):
```ruby
association.info = auth_token[:info] || {}
association.credentials = auth_token[:credentials] || {}
association.extra = auth_token[:extra] || {}
association.last_used = Time.zone.now
association.save!
```

**Current Code:**
```ruby
UserAssociatedAccount.create!(
  provider_name: 'oidc',
  provider_uid: @user_info[:sub],
  user_id: user.id,
  extra: { ... }.to_json  # ❌ Only sets extra, missing info/credentials/last_used
)
```

**Required Fix:**
```ruby
UserAssociatedAccount.create!(
  provider_name: 'oidc',
  provider_uid: @user_info[:sub],
  user_id: user.id,
  info: {
    email: @user_info[:email],
    name: @user_info[:name],
    picture: @user_info[:picture]
  },
  credentials: {},  # Empty for now, could store token metadata later
  extra: {
    email_verified: @user_info[:email_verified],
    created_via: 'mobile_session_exchange'
  },
  last_used: Time.zone.now
)
```

---

## Missing Features

### ⚠️ MISSING #1: Staged User Conversion

**Official Implementation** (`users_controller.rb:690-695`):
```ruby
user = User.where(staged: true).with_email(new_user_params[:email].strip.downcase).first

if user
  user.active = false
  user.unstage!  # Converts staged user to real user
end
```

**Impact:** If a user was invited to Discourse (created as "staged"), they won't be properly converted when signing up via mobile.

**Fix for `find_existing_user`:**
```ruby
def find_existing_user
  email = @user_info[:email].downcase.strip

  # First check for regular users (including staged)
  user = User.with_email(email).first

  # If staged, convert to real user
  if user&.staged?
    user.active = false
    user.unstage!
    user.save!
  end

  return user if user

  # Fallback: check by logto_sub custom field
  user_id = UserCustomField.where(name: 'logto_sub', value: @user_info[:sub]).first&.user_id
  user_id ? User.find_by(id: user_id) : nil
end
```

---

### ⚠️ MISSING #2: Profile Sync (Bio, Location)

**Official Implementation** (`managed_authenticator.rb:147-159`):
```ruby
def retrieve_profile(user, info)
  return unless user

  bio = info["description"]
  location = info["location"]

  if bio || location
    profile = user.user_profile
    profile.bio_raw = bio if profile.bio_raw.blank?
    profile.location = location if profile.location.blank?
    profile.save
  end
end
```

**Impact:** User bio/location from Logto won't sync to Discourse profile.

**Fix:** Add to `create_new_user` and `update_user_info`:
```ruby
def sync_profile(user)
  return unless @user_info[:bio].present? || @user_info[:location].present?

  profile = user.user_profile
  profile.bio_raw = @user_info[:bio] if profile.bio_raw.blank? && @user_info[:bio].present?
  profile.location = @user_info[:location] if profile.location.blank? && @user_info[:location].present?
  profile.save
end
```

---

### ⚠️ MISSING #3: Avatar Override Respect

**Current Implementation:** ✓ Has avatar retrieval (line 79-85)

**Missing:** Doesn't respect `SiteSetting.auth_overrides_avatar`

**Official Pattern** (`managed_authenticator.rb:141-145`):
```ruby
def retrieve_avatar(user, url)
  return unless user && url.present?
  return if user.user_avatar.try(:custom_upload_id).present? && !SiteSetting.auth_overrides_avatar
  Jobs.enqueue(:download_avatar_from_url, ...)
end
```

**Fix at line 79:**
```ruby
if @user_info[:picture].present?
  # Don't override if user has custom avatar and setting disallows it
  unless user.user_avatar&.custom_upload_id.present? && !SiteSetting.auth_overrides_avatar
    Jobs.enqueue(:download_avatar_from_url,
      url: @user_info[:picture],
      user_id: user.id,
      override_gravatar: false
    )
  end
end
```

---

## Architecture Comparison

### Official OIDC Flow (Web)

```
1. User clicks "Login with OIDC"
2. OAuth redirect to provider
3. Provider redirects to /auth/oidc/callback
4. OmniauthCallbacksController receives auth_hash
5. OpenIDConnectAuthenticator.after_authenticate(auth_hash)
   ├─ Find/init association by (provider, uid)
   ├─ Match by email if enabled
   ├─ Update association metadata (info, credentials, extra, last_used)
   └─ Return Auth::Result
6. If user found: log in
7. If not found: store auth in session → user registers → after_create_account
```

### Your Plugin Flow (Mobile)

```
1. Mobile app gets Logto access token
2. POST /api/auth/mobile-session with token
3. TokenValidator validates token → userinfo
4. UserProvisioner.provision(userinfo)
   ├─ find_existing_user (by email or logto_sub)
   ├─ If found: update_user_info ❌ (doesn't ensure association)
   └─ If not: create_new_user ❌ (doesn't cleanup old associations)
5. SessionManager.create_session
6. Return cookies
```

**Key Difference:** You bypass Discourse's normal user creation flow, which means:
- ✅ Simpler, one-step process
- ✅ No session storage needed
- ❌ Bypasses some validations (invite codes, custom user fields)
- ❌ Must manually handle all edge cases

---

## Recommended Refactor

### Option 1: Fix Current Implementation (Recommended)

Apply all fixes above to make your implementation robust while keeping the current architecture.

**Files to modify:**
- `user_provisioner.rb` - Apply all 3 bug fixes + missing features

**Estimated effort:** 1-2 hours

---

### Option 2: Use Official Authenticator

Refactor to use `Auth::ManagedAuthenticator` directly:

```ruby
# In UserProvisioner or new service class
def provision_via_official_authenticator(user_info)
  # Build auth_hash compatible with official OIDC
  auth_hash = {
    provider: 'oidc',
    uid: user_info[:sub],
    info: {
      email: user_info[:email],
      name: user_info[:name],
      image: user_info[:picture]
    },
    extra: {
      raw_info: user_info  # Must include email_verified
    },
    credentials: {}
  }

  authenticator = Discourse.enabled_authenticators.find { |a| a.name == 'oidc' }
  auth_result = authenticator.after_authenticate(auth_hash)

  # If user already exists, return it
  return auth_result.user if auth_result.user

  # Otherwise create user (simplified version of UsersController#create)
  user = create_user_from_auth_result(auth_result)

  # Link association
  authenticator.after_create_account(user, auth_result)

  user
end
```

**Benefits:**
- ✅ 100% compatible with official OIDC
- ✅ Automatic future-proofing
- ✅ Less code to maintain

**Drawbacks:**
- ⚠️ Requires understanding Auth::Result flow
- ⚠️ More refactoring needed

**Estimated effort:** 3-4 hours

---

## Testing Checklist

After applying fixes, test these scenarios:

- [ ] **New user signup** - Creates user + association
- [ ] **Existing user (no association)** - Finds user + creates association
- [ ] **Existing user (with association)** - Finds user + updates association
- [ ] **User switches Logto accounts (same email)** - Destroys old association + creates new
- [ ] **Staged user conversion** - Converts staged → real user
- [ ] **Avatar sync** - Downloads avatar on first login
- [ ] **Profile sync** - Syncs bio/location if present
- [ ] **Duplicate prevention** - Doesn't crash on constraint violations

---

## Quick Fix Summary

**Priority 1 (CRITICAL - Fix Immediately):**
1. Add `destroy_all` before creating UserAssociatedAccount (Bug #1)
2. Add association handling in `update_user_info` (Bug #2)
3. Populate all UserAssociatedAccount fields (Bug #3)

**Priority 2 (Important - Fix Soon):**
4. Handle staged user conversion
5. Add profile sync

**Priority 3 (Nice to Have):**
6. Respect `auth_overrides_avatar` setting

---

## Code Review Recommendations

Based on official Discourse patterns:

1. **Use `find_or_initialize_by` pattern** instead of separate find + create
2. **Always populate JSONB fields** even if empty (`{}` not `nil`)
3. **Use `Time.zone.now`** not `Time.now` for timestamps
4. **Handle ActiveRecord::RecordNotUnique** explicitly
5. **Follow official field naming** (use `info`, `credentials`, `extra` as intended)

---

## Conclusion

Your implementation is **70% correct** but has critical edge case bugs. The official OIDC implementation has battle-tested patterns for handling:

- Duplicate associations (via `destroy_all`)
- Partial user data (via `find_or_initialize_by`)
- Association lifecycle (via separate `after_authenticate` and `after_create_account`)

**Bottom line:** Either apply the fixes above OR refactor to use the official authenticator directly. Don't ship to production without addressing Bug #1 and Bug #2.
