### Task 3: Client `CommentsRepository`

**Repo:** `client-app`

**Files:**
- Create: `auth/src/commonMain/kotlin/pro/masterdoc/client/auth/CommentsRepository.kt`
- Create: `auth/src/jvmTest/kotlin/pro/masterdoc/client/auth/CommentsRepositoryTest.kt` (jvmTest RecordingGatewayHttpClient)

**Interfaces:**
```kotlin
@Serializable
data class WorkOrderCommentDto(
    val id: String,
    val orgId: String,
    val workOrderId: String,
    val authorId: String,
    val text: String,
    val attachmentId: String? = null,
    val createdAt: String,
)

@Serializable
data class CreateWorkOrderCommentRequest(
    val workOrderId: String,
    val text: String,
    val attachmentId: String? = null,
)

class CommentsRepository(...) {
    suspend fun list(workOrderId: String): List<WorkOrderCommentDto>
    suspend fun create(request: CreateWorkOrderCommentRequest): WorkOrderCommentDto
}
```

- [ ] **Step 1:** Failing JVM tests for GET query + POST body.
- [ ] **Step 2:** Implement.
- [ ] **Step 3:** Commit `feat(auth): CommentsRepository for work order feed`.

Work from: `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/client-app`

API paths:
- `GET {gateway}/comments?workOrderId=`
- `POST {gateway}/comments` JSON body
