# Organizations Module - Developer Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Database Schema](#database-schema)
4. [File Structure](#file-structure)
5. [Development Sequence](#development-sequence)
6. [API Endpoints](#api-endpoints)
7. [DTOs (Data Transfer Objects)](#dtos-data-transfer-objects)
8. [Service Methods](#service-methods)
9. [Authentication & Authorization](#authentication--authorization)
10. [Error Handling](#error-handling)
11. [Testing](#testing)
12. [Common Workflows](#common-workflows)

---

## Overview

The Organizations module is the foundation of SQDIS's multi-tenant architecture. It manages:
- Organization creation and settings
- Member invitations and management
- Role-based access control (RBAC)
- Organization-level data isolation

### Key Features
- Multi-tenant organization support
- Email-based invitation system with 7-day expiry
- Four-tier role hierarchy (OWNER, ADMIN, TEAM_LEAD, DEVELOPER)
- Audit logging for all critical operations
- Cascade deletion of organization data

### Role Hierarchy
```
OWNER       → Full control (delete org, manage all members)
ADMIN       → Manage members, update settings
TEAM_LEAD   → Read access, team management
DEVELOPER   → Read access only
```

---

## Architecture

### Module Dependencies
```
OrganizationsModule
├── PrismaModule (database access)
├── AuthModule (JWT guards, role guards)
└── AuditModule (audit logging)
```

### Request Flow
```
Client Request
    ↓
Controller (validation, auth guards)
    ↓
Service (business logic)
    ↓
Prisma (PostgreSQL)
    ↓
Response
```

---

## Database Schema

### Organization Table
```prisma
model Organization {
  id        String   @id @default(uuid())
  name      String
  slug      String   @unique
  logoUrl   String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  // Relations
  members               OrganizationMember[]
  invitations           Invitation[]
  githubConnections     GitHubConnection[]
  repositories          Repository[]
  teams                 Team[]
  projects              Project[]
  // ... and 15+ more relations
}
```

### OrganizationMember Table
```prisma
model OrganizationMember {
  id             String   @id @default(uuid())
  organizationId String
  userId         String
  role           Role     @default(DEVELOPER)
  joinedAt       DateTime @default(now())
  
  organization Organization @relation(...)
  user         User         @relation(...)
  
  @@unique([organizationId, userId])
}
```

### Invitation Table
```prisma
model Invitation {
  id             String    @id @default(uuid())
  organizationId String
  email          String
  token          String    @unique
  expiresAt      DateTime
  acceptedAt     DateTime?
  createdAt      DateTime  @default(now())
  
  organization Organization @relation(...)
}
```

### Role Enum
```prisma
enum Role {
  OWNER
  ADMIN
  TEAM_LEAD
  DEVELOPER
}
```

---

## File Structure

```
backend/src/modules/organizations/
├── dto/
│   ├── create-organization.dto.ts
│   ├── update-organization.dto.ts
│   ├── invite-member.dto.ts
│   ├── update-member-role.dto.ts
│   └── index.ts
├── organizations.controller.ts
├── organizations.service.ts
├── organizations.module.ts
├── organizations.service.spec.ts
└── index.ts
```


---

## Development Sequence

When building or modifying the Organizations module, follow this sequence:

### 1. Database Schema (Prisma)
**File:** `backend/prisma/schema.prisma`
- Define models: Organization, OrganizationMember, Invitation
- Define enums: Role
- Set up relations and constraints
- Run migration: `npx prisma migrate dev`

### 2. DTOs (Data Transfer Objects)
**Order:**
1. `create-organization.dto.ts` - For creating organizations
2. `update-organization.dto.ts` - For updating settings
3. `invite-member.dto.ts` - For inviting members
4. `update-member-role.dto.ts` - For role changes

**Key Points:**
- Use class-validator decorators (@IsString, @IsEmail, etc.)
- Add Swagger decorators (@ApiProperty)
- Include validation messages
- Keep DTOs minimal (only required fields)

### 3. Service Layer
**File:** `organizations.service.ts`

**Implementation Order:**
1. Basic CRUD operations
   - `create()` - Create organization
   - `findById()` - Get by ID
   - `findBySlug()` - Get by slug
   - `update()` - Update settings
   - `delete()` - Delete organization

2. Member management
   - `getMembers()` - List members
   - `isUserMember()` - Check membership
   - `getUserRole()` - Get user's role
   - `verifyUserRole()` - Authorization check

3. Invitation system
   - `createInvitation()` - Create invitation
   - `acceptInvitation()` - Accept invitation
   - `resendInvitation()` - Resend invitation
   - `getInvitationByToken()` - Get invitation details

4. Advanced operations
   - `updateMemberRole()` - Change member role
   - `removeMember()` - Remove member
   - `findAllForUser()` - Get user's organizations

### 4. Controller Layer
**File:** `organizations.controller.ts`

**Implementation Order:**
1. Organization CRUD endpoints
2. Member listing endpoints
3. Invitation endpoints
4. Member management endpoints

**Key Points:**
- Apply guards (@UseGuards)
- Add Swagger documentation (@ApiOperation, @ApiResponse)
- Use decorators (@GetUser, @Roles)
- Add audit logging (@AuditLog)

### 5. Module Configuration
**File:** `organizations.module.ts`
- Import dependencies (PrismaModule, AuthModule, AuditModule)
- Register controller and service
- Export service for other modules

### 6. Testing
**File:** `organizations.service.spec.ts`
- Unit tests for service methods
- Mock Prisma client
- Test error scenarios
- Test authorization logic


---

## API Endpoints

### Organization Management

#### 1. Create Organization
```http
POST /api/organizations
Authorization: Bearer <jwt-token>
Content-Type: application/json

{
  "name": "Acme Corporation",
  "slug": "acme-corp"
}
```

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "name": "Acme Corporation",
  "slug": "acme-corp",
  "logoUrl": null,
  "createdAt": "2024-01-01T00:00:00.000Z",
  "updatedAt": "2024-01-01T00:00:00.000Z"
}
```

**Errors:**
- `409 Conflict` - Slug already exists
- `400 Bad Request` - Validation error

**Business Logic:**
- Creator automatically becomes OWNER
- Slug is converted to lowercase
- Slug must be unique across all organizations

---

#### 2. Get All Organizations (for current user)
```http
GET /api/organizations
Authorization: Bearer <jwt-token>
```

**Response:** `200 OK`
```json
[
  {
    "id": "uuid",
    "name": "Acme Corporation",
    "slug": "acme-corp",
    "logoUrl": null,
    "createdAt": "2024-01-01T00:00:00.000Z",
    "updatedAt": "2024-01-01T00:00:00.000Z",
    "memberCount": 5
  }
]
```

---

#### 3. Get Organization by ID
```http
GET /api/organizations/:id
Authorization: Bearer <jwt-token>
```

**Response:** `200 OK`
```json
{
  "id": "uuid",
  "name": "Acme Corporation",
  "slug": "acme-corp",
  "logoUrl": "https://example.com/logo.png",
  "createdAt": "2024-01-01T00:00:00.000Z",
  "updatedAt": "2024-01-01T00:00:00.000Z"
}
```

**Errors:**
- `403 Forbidden` - User is not a member
- `404 Not Found` - Organization doesn't exist

**Authorization:** Any member (OWNER, ADMIN, TEAM_LEAD, DEVELOPER)

---

#### 4. Update Organization
```http
PATCH /api/organizations/:id
Authorization: Bearer <jwt-token>
Content-Type: application/json

{
  "name": "Acme Corp Updated",
  "logoUrl": "https://example.com/new-logo.png"
}
```

**Response:** `200 OK`
```json
{
  "id": "uuid",
  "name": "Acme Corp Updated",
  "slug": "acme-corp",
  "logoUrl": "https://example.com/new-logo.png",
  "createdAt": "2024-01-01T00:00:00.000Z",
  "updatedAt": "2024-01-02T00:00:00.000Z"
}
```

**Errors:**
- `403 Forbidden` - User is not OWNER or ADMIN
- `404 Not Found` - Organization doesn't exist
- `409 Conflict` - New slug already exists

**Authorization:** OWNER, ADMIN only


---

#### 5. Delete Organization
```http
DELETE /api/organizations/:id
Authorization: Bearer <jwt-token>
```

**Response:** `204 No Content`

**Errors:**
- `403 Forbidden` - User is not OWNER
- `404 Not Found` - Organization doesn't exist

**Authorization:** OWNER only

**Important:** Cascade deletes ALL related data:
- Members
- Invitations
- GitHub connections
- Repositories
- Teams
- Projects
- Commits
- Scores
- And all other organization data

---

### Member Management

#### 6. Get Organization Members
```http
GET /api/organizations/:id/members
Authorization: Bearer <jwt-token>
```

**Response:** `200 OK`
```json
[
  {
    "id": "membership-uuid",
    "userId": "user-uuid",
    "role": "OWNER",
    "joinedAt": "2024-01-01T00:00:00.000Z",
    "user": {
      "id": "user-uuid",
      "email": "owner@example.com",
      "name": "John Doe",
      "avatarUrl": "https://example.com/avatar.png"
    }
  },
  {
    "id": "membership-uuid-2",
    "userId": "user-uuid-2",
    "role": "DEVELOPER",
    "joinedAt": "2024-01-02T00:00:00.000Z",
    "user": {
      "id": "user-uuid-2",
      "email": "dev@example.com",
      "name": "Jane Smith",
      "avatarUrl": null
    }
  }
]
```

**Errors:**
- `403 Forbidden` - User is not a member
- `404 Not Found` - Organization doesn't exist

**Authorization:** Any member

---

#### 7. Invite Member
```http
POST /api/organizations/:id/invite
Authorization: Bearer <jwt-token>
Content-Type: application/json

{
  "email": "newmember@example.com"
}
```

**Response:** `201 Created`
```json
{
  "id": "invitation-uuid",
  "email": "newmember@example.com",
  "token": "64-char-hex-token",
  "expiresAt": "2024-01-08T00:00:00.000Z",
  "createdAt": "2024-01-01T00:00:00.000Z",
  "acceptedAt": null,
  "organizationId": "org-uuid"
}
```

**Errors:**
- `403 Forbidden` - User is not OWNER or ADMIN
- `404 Not Found` - Organization doesn't exist
- `409 Conflict` - User already a member or invitation exists

**Authorization:** OWNER, ADMIN only

**Business Logic:**
- Token expires in 7 days
- Email is converted to lowercase
- Cannot invite existing members
- Only one pending invitation per email


---

#### 8. Get Invitation Details
```http
GET /api/organizations/invitations/:token
Authorization: Bearer <jwt-token>
```

**Response:** `200 OK`
```json
{
  "id": "invitation-uuid",
  "email": "newmember@example.com",
  "token": "64-char-hex-token",
  "expiresAt": "2024-01-08T00:00:00.000Z",
  "createdAt": "2024-01-01T00:00:00.000Z",
  "acceptedAt": null,
  "organizationId": "org-uuid",
  "organization": {
    "id": "org-uuid",
    "name": "Acme Corporation",
    "slug": "acme-corp",
    "logoUrl": null,
    "createdAt": "2024-01-01T00:00:00.000Z",
    "updatedAt": "2024-01-01T00:00:00.000Z"
  }
}
```

**Errors:**
- `404 Not Found` - Invitation doesn't exist

---

#### 9. Accept Invitation
```http
POST /api/organizations/invitations/:token/accept
Authorization: Bearer <jwt-token>
```

**Response:** `201 Created`
```json
{
  "id": "membership-uuid",
  "userId": "user-uuid",
  "role": "DEVELOPER",
  "joinedAt": "2024-01-02T00:00:00.000Z",
  "user": {
    "id": "user-uuid",
    "email": "newmember@example.com",
    "name": "New Member",
    "avatarUrl": null
  }
}
```

**Errors:**
- `400 Bad Request` - Invitation already accepted
- `404 Not Found` - Invitation doesn't exist
- `409 Conflict` - User already a member
- `410 Gone` - Invitation expired

**Business Logic:**
- New members get DEVELOPER role by default
- Invitation is marked as accepted (acceptedAt timestamp)
- Cannot accept expired invitations
- Cannot accept already-accepted invitations

---

#### 10. Resend Invitation
```http
POST /api/organizations/:id/invite/resend
Authorization: Bearer <jwt-token>
Content-Type: application/json

{
  "email": "newmember@example.com"
}
```

**Response:** `201 Created`
```json
{
  "id": "invitation-uuid",
  "email": "newmember@example.com",
  "token": "new-64-char-hex-token",
  "expiresAt": "2024-01-09T00:00:00.000Z",
  "createdAt": "2024-01-01T00:00:00.000Z",
  "acceptedAt": null,
  "organizationId": "org-uuid"
}
```

**Errors:**
- `403 Forbidden` - User is not OWNER or ADMIN
- `404 Not Found` - No pending invitation found

**Authorization:** OWNER, ADMIN only

**Business Logic:**
- Generates new token
- Resets expiry to 7 days from now
- Updates existing invitation (doesn't create new one)


---

#### 11. Update Member Role
```http
PATCH /api/organizations/:id/members/:userId
Authorization: Bearer <jwt-token>
Content-Type: application/json

{
  "role": "ADMIN"
}
```

**Response:** `200 OK`
```json
{
  "id": "membership-uuid",
  "userId": "user-uuid",
  "role": "ADMIN",
  "joinedAt": "2024-01-02T00:00:00.000Z",
  "user": {
    "id": "user-uuid",
    "email": "member@example.com",
    "name": "Member Name",
    "avatarUrl": null
  }
}
```

**Errors:**
- `400 Bad Request` - Cannot demote last OWNER
- `403 Forbidden` - User is not OWNER or trying to change own role
- `404 Not Found` - Organization or member not found

**Authorization:** OWNER only

**Business Logic:**
- Cannot change your own role
- Cannot demote the last OWNER (must assign another OWNER first)
- Role change is logged to audit log

---

#### 12. Remove Member
```http
DELETE /api/organizations/:id/members/:userId
Authorization: Bearer <jwt-token>
```

**Response:** `204 No Content`

**Errors:**
- `400 Bad Request` - Cannot remove last OWNER
- `403 Forbidden` - User is not OWNER/ADMIN or trying to remove self
- `404 Not Found` - Organization or member not found

**Authorization:** OWNER, ADMIN only

**Business Logic:**
- Cannot remove yourself (use leave organization instead)
- Cannot remove the last OWNER
- Cascade deletes related data (team memberships, etc.)

---

## DTOs (Data Transfer Objects)

### CreateOrganizationDto
**File:** `dto/create-organization.dto.ts`

```typescript
export class CreateOrganizationDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(1)
  @MaxLength(100)
  name: string;

  @IsString()
  @IsNotEmpty()
  @MinLength(3)
  @MaxLength(50)
  @Matches(/^[a-z0-9]+(?:-[a-z0-9]+)*$/)
  slug: string;
}
```

**Validation Rules:**
- `name`: 1-100 characters, required
- `slug`: 3-50 characters, lowercase alphanumeric with hyphens, required

**Example:**
```json
{
  "name": "Acme Corporation",
  "slug": "acme-corp"
}
```

---

### UpdateOrganizationDto
**File:** `dto/update-organization.dto.ts`

```typescript
export class UpdateOrganizationDto {
  @IsOptional()
  @IsString()
  @MinLength(1)
  @MaxLength(100)
  name?: string;

  @IsOptional()
  @IsString()
  @MinLength(3)
  @MaxLength(50)
  @Matches(/^[a-z0-9]+(?:-[a-z0-9]+)*$/)
  slug?: string;

  @IsOptional()
  @IsString()
  @IsUrl()
  logoUrl?: string;
}
```

**Validation Rules:**
- All fields optional
- `name`: 1-100 characters
- `slug`: 3-50 characters, lowercase alphanumeric with hyphens
- `logoUrl`: Valid URL

**Example:**
```json
{
  "name": "Acme Corp Updated",
  "logoUrl": "https://example.com/logo.png"
}
```


---

### InviteMemberDto
**File:** `dto/invite-member.dto.ts`

```typescript
export class InviteMemberDto {
  @IsEmail()
  @IsNotEmpty()
  email: string;
}
```

**Validation Rules:**
- `email`: Valid email format, required

**Example:**
```json
{
  "email": "newmember@example.com"
}
```

---

### UpdateMemberRoleDto
**File:** `dto/update-member-role.dto.ts`

```typescript
export class UpdateMemberRoleDto {
  @IsEnum(Role)
  @IsNotEmpty()
  role: Role;
}
```

**Validation Rules:**
- `role`: Must be one of: OWNER, ADMIN, TEAM_LEAD, DEVELOPER

**Example:**
```json
{
  "role": "ADMIN"
}
```

---

## Service Methods

### Core CRUD Operations

#### create(dto, ownerId)
**Purpose:** Create a new organization

**Parameters:**
- `dto: CreateOrganizationDto` - Organization data
- `ownerId: string` - User ID of creator

**Returns:** `Promise<OrganizationResponse>`

**Logic:**
1. Check if slug already exists (throw ConflictException)
2. Convert slug to lowercase
3. Create organization
4. Create membership with OWNER role
5. Return organization data

**Throws:**
- `ConflictException` - Slug already exists

---

#### findById(id)
**Purpose:** Get organization by ID

**Parameters:**
- `id: string` - Organization ID

**Returns:** `Promise<OrganizationResponse>`

**Throws:**
- `NotFoundException` - Organization not found

---

#### findBySlug(slug)
**Purpose:** Get organization by slug

**Parameters:**
- `slug: string` - Organization slug

**Returns:** `Promise<OrganizationResponse>`

**Throws:**
- `NotFoundException` - Organization not found

---

#### update(id, dto)
**Purpose:** Update organization settings

**Parameters:**
- `id: string` - Organization ID
- `dto: UpdateOrganizationDto` - Update data

**Returns:** `Promise<OrganizationResponse>`

**Logic:**
1. Check if organization exists
2. If slug is being updated, check uniqueness
3. Convert slug to lowercase
4. Update organization
5. Return updated data

**Throws:**
- `NotFoundException` - Organization not found
- `ConflictException` - New slug already exists

---

#### delete(id)
**Purpose:** Delete organization (cascade deletes all data)

**Parameters:**
- `id: string` - Organization ID

**Returns:** `Promise<void>`

**Throws:**
- `NotFoundException` - Organization not found

**Warning:** This permanently deletes ALL organization data including:
- Members, invitations, repositories, commits, scores, teams, projects, etc.


---

### Member Management

#### findAllForUser(userId)
**Purpose:** Get all organizations for a user

**Parameters:**
- `userId: string` - User ID

**Returns:** `Promise<OrganizationWithMemberCount[]>`

**Logic:**
1. Find all memberships for user
2. Include organization data
3. Include member count
4. Return array of organizations

---

#### getMembers(organizationId)
**Purpose:** Get all members of an organization

**Parameters:**
- `organizationId: string` - Organization ID

**Returns:** `Promise<OrganizationMemberResponse[]>`

**Logic:**
1. Check if organization exists
2. Find all memberships
3. Include user data (email, name, avatarUrl)
4. Order by joinedAt (ascending)
5. Return array of members

**Throws:**
- `NotFoundException` - Organization not found

---

#### isUserMember(organizationId, userId)
**Purpose:** Check if user is a member

**Parameters:**
- `organizationId: string` - Organization ID
- `userId: string` - User ID

**Returns:** `Promise<boolean>`

**Logic:**
1. Query membership table
2. Return true if membership exists, false otherwise

---

#### getUserRole(organizationId, userId)
**Purpose:** Get user's role in organization

**Parameters:**
- `organizationId: string` - Organization ID
- `userId: string` - User ID

**Returns:** `Promise<Role | null>`

**Logic:**
1. Query membership table
2. Return role if membership exists, null otherwise

---

#### verifyUserRole(organizationId, userId, requiredRoles)
**Purpose:** Verify user has required role (throws exception if not)

**Parameters:**
- `organizationId: string` - Organization ID
- `userId: string` - User ID
- `requiredRoles: Role[]` - Array of acceptable roles

**Returns:** `Promise<void>`

**Logic:**
1. Get user's role
2. If no role, throw ForbiddenException (not a member)
3. If role not in requiredRoles, throw ForbiddenException (insufficient permissions)

**Throws:**
- `ForbiddenException` - User not a member or insufficient permissions

**Usage Example:**
```typescript
// Only allow OWNER and ADMIN
await this.organizationsService.verifyUserRole(
  organizationId,
  userId,
  [Role.OWNER, Role.ADMIN]
);
```


---

### Invitation System

#### createInvitation(organizationId, email)
**Purpose:** Create invitation with 7-day expiry

**Parameters:**
- `organizationId: string` - Organization ID
- `email: string` - Email to invite

**Returns:** `Promise<InvitationResponse>`

**Logic:**
1. Check if organization exists
2. Convert email to lowercase
3. Check if user already a member (throw ConflictException)
4. Check for existing pending invitation (throw ConflictException)
5. Generate secure 64-char hex token
6. Set expiry to 7 days from now
7. Create invitation
8. Return invitation data

**Throws:**
- `NotFoundException` - Organization not found
- `ConflictException` - User already member or invitation exists

---

#### acceptInvitation(token, userId)
**Purpose:** Accept invitation and add user to organization

**Parameters:**
- `token: string` - Invitation token
- `userId: string` - User ID accepting invitation

**Returns:** `Promise<OrganizationMemberResponse>`

**Logic:**
1. Find invitation by token
2. Check if already accepted (throw BadRequestException)
3. Check if expired (throw GoneException)
4. Check if user already a member (throw ConflictException)
5. Create membership with DEVELOPER role
6. Mark invitation as accepted
7. Return membership data

**Throws:**
- `NotFoundException` - Invitation not found
- `BadRequestException` - Already accepted
- `GoneException` - Invitation expired
- `ConflictException` - User already member

---

#### resendInvitation(organizationId, email)
**Purpose:** Resend invitation with new token and expiry

**Parameters:**
- `organizationId: string` - Organization ID
- `email: string` - Email to resend to

**Returns:** `Promise<InvitationResponse>`

**Logic:**
1. Find existing pending invitation
2. Generate new token
3. Set new expiry (7 days from now)
4. Update invitation
5. Return updated invitation data

**Throws:**
- `NotFoundException` - No pending invitation found

---

#### getInvitationByToken(token)
**Purpose:** Get invitation details for acceptance flow

**Parameters:**
- `token: string` - Invitation token

**Returns:** `Promise<InvitationResponse & { organization: OrganizationResponse }>`

**Logic:**
1. Find invitation by token
2. Include organization data
3. Return invitation with organization

**Throws:**
- `NotFoundException` - Invitation not found


---

### Advanced Operations

#### updateMemberRole(organizationId, targetUserId, newRole, requestingUserId)
**Purpose:** Change member's role

**Parameters:**
- `organizationId: string` - Organization ID
- `targetUserId: string` - User ID to update
- `newRole: Role` - New role
- `requestingUserId: string` - User ID making the change

**Returns:** `Promise<OrganizationMemberResponse>`

**Logic:**
1. Check if organization exists
2. Find target membership
3. Prevent changing own role (throw ForbiddenException)
4. If demoting last OWNER, throw BadRequestException
5. Update role
6. Log to audit log
7. Return updated membership

**Throws:**
- `NotFoundException` - Organization or member not found
- `ForbiddenException` - Cannot change own role
- `BadRequestException` - Cannot demote last OWNER

---

#### removeMember(organizationId, targetUserId, requestingUserId)
**Purpose:** Remove member from organization

**Parameters:**
- `organizationId: string` - Organization ID
- `targetUserId: string` - User ID to remove
- `requestingUserId: string` - User ID making the change

**Returns:** `Promise<void>`

**Logic:**
1. Check if organization exists
2. Find target membership
3. Prevent removing self (throw ForbiddenException)
4. If removing last OWNER, throw BadRequestException
5. Delete membership

**Throws:**
- `NotFoundException` - Organization or member not found
- `ForbiddenException` - Cannot remove self
- `BadRequestException` - Cannot remove last OWNER

---

## Authentication & Authorization

### Guards Used

#### JwtAuthGuard
**Purpose:** Verify JWT token and extract user

**Applied to:** All endpoints (controller level)

**Usage:**
```typescript
@UseGuards(JwtAuthGuard)
@Controller('organizations')
export class OrganizationsController {}
```

---

#### RolesGuard
**Purpose:** Verify user has required role

**Applied to:** Specific endpoints requiring role checks

**Usage:**
```typescript
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(Role.OWNER, Role.ADMIN)
@Post(':id/invite')
async inviteMember() {}
```

---

### Decorators

#### @GetUser(field?)
**Purpose:** Extract user data from request

**Usage:**
```typescript
async create(@GetUser('id') userId: string) {}
```

---

#### @Roles(...roles)
**Purpose:** Specify required roles for endpoint

**Usage:**
```typescript
@Roles(Role.OWNER)
@Patch(':id/members/:userId')
async updateMemberRole() {}
```

---

#### @AuditLog(config)
**Purpose:** Log action to audit trail

**Usage:**
```typescript
@AuditLog({
  action: 'CREATE',
  resourceType: 'Organization',
  captureSnapshot: true,
  includeRequestBody: true,
  includeResponseBody: true,
})
@Post()
async create() {}
```


---

### Authorization Matrix

| Endpoint | OWNER | ADMIN | TEAM_LEAD | DEVELOPER |
|----------|-------|-------|-----------|-----------|
| Create Organization | ✅ | ✅ | ✅ | ✅ |
| Get Organizations | ✅ | ✅ | ✅ | ✅ |
| Get Organization | ✅ | ✅ | ✅ | ✅ |
| Update Organization | ✅ | ✅ | ❌ | ❌ |
| Delete Organization | ✅ | ❌ | ❌ | ❌ |
| Get Members | ✅ | ✅ | ✅ | ✅ |
| Invite Member | ✅ | ✅ | ❌ | ❌ |
| Accept Invitation | ✅ | ✅ | ✅ | ✅ |
| Resend Invitation | ✅ | ✅ | ❌ | ❌ |
| Update Member Role | ✅ | ❌ | ❌ | ❌ |
| Remove Member | ✅ | ✅ | ❌ | ❌ |

---

## Error Handling

### HTTP Status Codes

| Code | Exception | When |
|------|-----------|------|
| 400 | BadRequestException | Validation error, cannot demote last OWNER |
| 401 | UnauthorizedException | Invalid/missing JWT token |
| 403 | ForbiddenException | Insufficient permissions |
| 404 | NotFoundException | Resource not found |
| 409 | ConflictException | Slug exists, user already member |
| 410 | GoneException | Invitation expired |

---

### Common Error Responses

#### 400 Bad Request
```json
{
  "statusCode": 400,
  "message": "Cannot demote the last owner. Assign another owner first.",
  "error": "Bad Request"
}
```

#### 403 Forbidden
```json
{
  "statusCode": 403,
  "message": "You do not have permission to perform this action",
  "error": "Forbidden"
}
```

#### 404 Not Found
```json
{
  "statusCode": 404,
  "message": "Organization with ID 'uuid' not found",
  "error": "Not Found"
}
```

#### 409 Conflict
```json
{
  "statusCode": 409,
  "message": "Organization with slug 'acme-corp' already exists",
  "error": "Conflict"
}
```

#### 410 Gone
```json
{
  "statusCode": 410,
  "message": "Invitation has expired. Please request a new invitation.",
  "error": "Gone"
}
```

---

## Testing

### Unit Tests
**File:** `organizations.service.spec.ts`

#### Test Structure
```typescript
describe('OrganizationsService', () => {
  let service: OrganizationsService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        OrganizationsService,
        {
          provide: PrismaService,
          useValue: mockPrismaService,
        },
        {
          provide: AuditLogService,
          useValue: mockAuditLogService,
        },
      ],
    }).compile();

    service = module.get<OrganizationsService>(OrganizationsService);
    prisma = module.get<PrismaService>(PrismaService);
  });

  // Tests here
});
```


---

#### Test Cases to Implement

**Organization CRUD:**
- ✅ Should create organization with valid data
- ✅ Should throw ConflictException if slug exists
- ✅ Should convert slug to lowercase
- ✅ Should assign creator as OWNER
- ✅ Should find organization by ID
- ✅ Should throw NotFoundException if not found
- ✅ Should update organization settings
- ✅ Should throw ConflictException if new slug exists
- ✅ Should delete organization

**Member Management:**
- ✅ Should get all members
- ✅ Should check if user is member
- ✅ Should get user role
- ✅ Should verify user role (success)
- ✅ Should throw ForbiddenException if not member
- ✅ Should throw ForbiddenException if insufficient role

**Invitations:**
- ✅ Should create invitation with 7-day expiry
- ✅ Should throw ConflictException if user already member
- ✅ Should throw ConflictException if invitation exists
- ✅ Should accept invitation
- ✅ Should throw GoneException if expired
- ✅ Should throw BadRequestException if already accepted
- ✅ Should resend invitation with new token

**Role Management:**
- ✅ Should update member role
- ✅ Should throw ForbiddenException if changing own role
- ✅ Should throw BadRequestException if demoting last OWNER
- ✅ Should log role change to audit

**Member Removal:**
- ✅ Should remove member
- ✅ Should throw ForbiddenException if removing self
- ✅ Should throw BadRequestException if removing last OWNER

---

### E2E Tests
**File:** `test/organizations.e2e-spec.ts`

```typescript
describe('Organizations (e2e)', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    // Setup test app
    // Login and get auth token
  });

  it('POST /organizations - should create organization', () => {
    return request(app.getHttpServer())
      .post('/organizations')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        name: 'Test Org',
        slug: 'test-org',
      })
      .expect(201)
      .expect((res) => {
        expect(res.body.name).toBe('Test Org');
        expect(res.body.slug).toBe('test-org');
      });
  });

  // More E2E tests...
});
```

---

## Common Workflows

### Workflow 1: Creating an Organization

**Steps:**
1. User registers/logs in
2. User calls `POST /api/organizations`
3. System validates input
4. System checks slug uniqueness
5. System creates organization
6. System creates membership with OWNER role
7. System returns organization data

**Code Example:**
```typescript
// Client side
const response = await fetch('/api/organizations', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    name: 'Acme Corporation',
    slug: 'acme-corp',
  }),
});

const organization = await response.json();
```

---

### Workflow 2: Inviting a Member

**Steps:**
1. OWNER/ADMIN calls `POST /api/organizations/:id/invite`
2. System validates email
3. System checks if user already member
4. System generates secure token
5. System creates invitation with 7-day expiry
6. System sends invitation email (handled by email service)
7. Invitee receives email with link
8. Invitee clicks link and calls `POST /api/organizations/invitations/:token/accept`
9. System validates token and expiry
10. System creates membership with DEVELOPER role
11. System marks invitation as accepted

**Code Example:**
```typescript
// Step 1: Invite
const invitation = await fetch(`/api/organizations/${orgId}/invite`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${adminToken}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    email: 'newmember@example.com',
  }),
});

// Step 2: Accept (by invitee)
const membership = await fetch(`/api/organizations/invitations/${token}/accept`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${inviteeToken}`,
  },
});
```


---

### Workflow 3: Promoting a Member

**Steps:**
1. OWNER calls `PATCH /api/organizations/:id/members/:userId`
2. System verifies requester is OWNER
3. System checks target member exists
4. System validates not changing own role
5. System validates not demoting last OWNER
6. System updates role
7. System logs change to audit log
8. System returns updated membership

**Code Example:**
```typescript
const updatedMember = await fetch(
  `/api/organizations/${orgId}/members/${userId}`,
  {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${ownerToken}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      role: 'ADMIN',
    }),
  }
);
```

---

### Workflow 4: Handling Expired Invitations

**Steps:**
1. User tries to accept expired invitation
2. System returns 410 Gone
3. User contacts OWNER/ADMIN
4. OWNER/ADMIN calls `POST /api/organizations/:id/invite/resend`
5. System generates new token with fresh expiry
6. System sends new invitation email
7. User accepts with new token

**Code Example:**
```typescript
// Try to accept
try {
  await fetch(`/api/organizations/invitations/${oldToken}/accept`, {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${token}` },
  });
} catch (error) {
  if (error.status === 410) {
    // Invitation expired - request resend
    console.log('Invitation expired. Please request a new one.');
  }
}

// Admin resends
await fetch(`/api/organizations/${orgId}/invite/resend`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${adminToken}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    email: 'newmember@example.com',
  }),
});
```

---

## Best Practices

### Security

1. **Always validate user permissions**
   - Use `verifyUserRole()` before sensitive operations
   - Don't trust client-side role checks

2. **Protect against enumeration attacks**
   - Return same error for "not found" and "no access"
   - Don't reveal if organization exists when user has no access

3. **Secure invitation tokens**
   - Use cryptographically secure random tokens (32 bytes)
   - Set reasonable expiry (7 days)
   - Mark as used after acceptance

4. **Prevent privilege escalation**
   - Users cannot change their own role
   - Must have at least one OWNER
   - Only OWNER can assign OWNER role

---

### Performance

1. **Use database indexes**
   - `organizationId_userId` unique index on memberships
   - `slug` unique index on organizations
   - `token` unique index on invitations

2. **Minimize database queries**
   - Use `include` to fetch related data in one query
   - Cache frequently accessed data (Redis)

3. **Paginate large lists**
   - Add pagination to member lists for large organizations
   - Use cursor-based pagination for better performance

---

### Code Quality

1. **Use TypeScript types**
   - Define response interfaces
   - Use Prisma-generated types
   - Avoid `any` type

2. **Write descriptive error messages**
   - Include context (resource ID, action)
   - Suggest solutions when possible

3. **Add JSDoc comments**
   - Document all public methods
   - Include parameter descriptions
   - Note thrown exceptions

4. **Follow naming conventions**
   - Use camelCase for methods
   - Use PascalCase for classes/interfaces
   - Use UPPER_CASE for enums


---

## Troubleshooting

### Common Issues

#### Issue: "Organization with slug already exists"
**Cause:** Slug is not unique
**Solution:** 
- Choose a different slug
- Check if organization was previously created
- Slugs are case-insensitive (converted to lowercase)

---

#### Issue: "Cannot demote the last owner"
**Cause:** Trying to change role of the only OWNER
**Solution:**
- First promote another member to OWNER
- Then demote the original OWNER

---

#### Issue: "Invitation has expired"
**Cause:** Token is older than 7 days
**Solution:**
- Use resend invitation endpoint
- New token will be generated with fresh expiry

---

#### Issue: "User does not have access to this organization"
**Cause:** User is not a member
**Solution:**
- Verify user has accepted invitation
- Check membership table
- Ensure invitation was sent to correct email

---

#### Issue: "Cannot remove yourself"
**Cause:** Trying to remove own membership
**Solution:**
- Use a dedicated "leave organization" endpoint (if implemented)
- Have another OWNER/ADMIN remove you

---

## Database Queries

### Useful SQL Queries for Debugging

#### Find all organizations for a user
```sql
SELECT o.* 
FROM organizations o
JOIN organization_members om ON o.id = om."organizationId"
WHERE om."userId" = 'user-uuid';
```

#### Find all members of an organization
```sql
SELECT u.email, u.name, om.role, om."joinedAt"
FROM organization_members om
JOIN users u ON om."userId" = u.id
WHERE om."organizationId" = 'org-uuid'
ORDER BY om."joinedAt" ASC;
```

#### Find pending invitations
```sql
SELECT *
FROM invitations
WHERE "organizationId" = 'org-uuid'
  AND "acceptedAt" IS NULL
  AND "expiresAt" > NOW();
```

#### Count OWNERs in organization
```sql
SELECT COUNT(*)
FROM organization_members
WHERE "organizationId" = 'org-uuid'
  AND role = 'OWNER';
```

#### Find expired invitations
```sql
SELECT *
FROM invitations
WHERE "expiresAt" < NOW()
  AND "acceptedAt" IS NULL;
```

---

## Integration with Other Modules

### Auth Module
**Dependency:** Organizations module uses Auth guards
- `JwtAuthGuard` - Validates JWT tokens
- `RolesGuard` - Checks user roles
- `@GetUser()` decorator - Extracts user from request

---

### Audit Module
**Dependency:** Organizations module logs to audit trail
- `@AuditLog()` decorator - Logs actions
- `AuditLogService.logRoleChange()` - Logs role changes

**Logged Actions:**
- Organization creation
- Organization updates
- Organization deletion
- Member invitations
- Role changes
- Member removal

---

### Teams Module
**Integration:** Teams belong to organizations
- Teams reference `organizationId`
- Team members must be organization members
- Deleting organization cascades to teams

---

### Projects Module
**Integration:** Projects belong to organizations
- Projects reference `organizationId`
- Deleting organization cascades to projects

---

### GitHub Module
**Integration:** GitHub connections belong to organizations
- One connection per organization
- Deleting organization cascades to GitHub data

---

## Environment Variables

No specific environment variables required for Organizations module.

Uses shared database connection:
```env
DATABASE_URL=postgresql://user:password@localhost:5432/sqdis
```

---

## API Documentation (Swagger)

Access Swagger UI at: `http://localhost:3000/api/docs`

**Organizations Tag:** All endpoints grouped under "Organizations"

**Try It Out:**
1. Click "Authorize" button
2. Enter JWT token: `Bearer <your-token>`
3. Click endpoint to expand
4. Click "Try it out"
5. Fill in parameters
6. Click "Execute"

---

## Quick Reference

### Key Files
- `organizations.controller.ts` - API endpoints
- `organizations.service.ts` - Business logic
- `organizations.module.ts` - Module configuration
- `dto/*.dto.ts` - Request validation
- `schema.prisma` - Database schema

### Key Endpoints
- `POST /organizations` - Create
- `GET /organizations` - List user's orgs
- `GET /organizations/:id` - Get details
- `PATCH /organizations/:id` - Update
- `DELETE /organizations/:id` - Delete
- `POST /organizations/:id/invite` - Invite member
- `POST /organizations/invitations/:token/accept` - Accept invitation

### Key Roles
- `OWNER` - Full control
- `ADMIN` - Manage members
- `TEAM_LEAD` - Read access
- `DEVELOPER` - Read access

### Key Exceptions
- `ConflictException` (409) - Duplicate slug/member
- `ForbiddenException` (403) - Insufficient permissions
- `NotFoundException` (404) - Resource not found
- `GoneException` (410) - Invitation expired

---

## Additional Resources

### Related Documentation
- [Auth Module Documentation](./auth.md)
- [Audit Module Documentation](./audit.md)
- [Prisma Documentation](https://www.prisma.io/docs)
- [NestJS Documentation](https://docs.nestjs.com)

### External Links
- [JWT Best Practices](https://tools.ietf.org/html/rfc8725)
- [RBAC Design Patterns](https://en.wikipedia.org/wiki/Role-based_access_control)
- [Multi-tenancy Patterns](https://docs.microsoft.com/en-us/azure/architecture/guide/multitenant/overview)

---

**Last Updated:** 2024-01-01  
**Module Version:** 1.0.0  
**Maintainer:** SQDIS Backend Team
