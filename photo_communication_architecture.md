# Software Architecture Design Document
## Photo Communication Platform for Manufacturing

**Document Version:** 1.0  
**Date:** December 2024  
**Author:** Architecture Team  
**Target Audience:** Engineering Leads and Managers  

---

## 1. Executive Summary

This document presents the architecture for a new photo-based communication platform designed to facilitate collaboration between mechanical engineers, procurement teams, and factory workers in large manufacturing companies (500+ employees). The platform enables real-time photo annotation and threaded discussions to address quality issues and streamline decision-making processes.

**Key Success Metrics:**
- Demonstrate platform viability and user value quickly
- Support 10,000+ concurrent users at scale
- 99.9% uptime SLA for manufacturing operations
- Sub-200ms API response times

---

## 2. System Architecture Overview

### 2.1 High-Level Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Web Frontend  │    │  Mobile App     │    │  Admin Portal   │
│   (React/TS)    │    │  (React Native) │    │  (React/TS)     │
└─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                    ┌─────────────▼─────────────┐
                    │    Load Balancer (GCP)    │
                    └─────────────┬─────────────┘
                                 │
                    ┌─────────────▼─────────────┐
                    │   API Gateway (Kong)      │
                    └─────────────┬─────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                       │                        │
┌───────▼───────┐    ┌──────────▼──────────┐    ┌───────▼───────┐
│  Auth Service  │    │  Photo Service      │    │ Comment Service│
│  (Ruby/Rails)  │    │  (TypeScript/Node)  │    │ (TypeScript)   │
└───────┬───────┘    └──────────┬──────────┘    └───────┬───────┘
        │                       │                        │
        │              ┌────────▼────────┐               │
        │              │  File Storage   │               │
        │              │  (GCS Buckets)  │               │
        │              └─────────────────┘               │
        │                                                │
        └────────────────────┬───────────────────────────┘
                            │
                ┌───────────▼───────────┐
                │   PostgreSQL DB       │
                │   (Cloud SQL)         │
                └───────────────────────┘
```

### 2.2 Technology Stack Alignment

**Frontend Technologies:**
- **TypeScript + React**: Aligns with CADDi's existing frontend stack
- **Next.js**: Server-side rendering for performance
- **Material-UI**: Consistent design system
- **WebSocket**: Real-time updates for annotations and comments

**Backend Technologies:**
- **Ruby on Rails**: Primary backend, leveraging CADDi's existing expertise
- **TypeScript/Node.js**: Microservices for real-time features
- **GraphQL**: Efficient data fetching for complex UI requirements
- **Redis**: Caching and session management

**Infrastructure:**
- **Google Kubernetes Engine (GKE)**: Container orchestration
- **Google Cloud SQL**: Managed PostgreSQL
- **Google Cloud Storage**: File storage with CDN
- **Istio Service Mesh**: Traffic management and security

---

## 3. Detailed Component Design

### 3.1 Core Services Architecture

**Authentication Service (Ruby on Rails)**
- Integrates with existing CADDi SSO
- Role-based access control (Engineer, Procurement, Factory Worker)
- JWT token management
- Audit logging for compliance

**Photo Management Service (TypeScript/Node.js)**
- Image upload/processing pipeline
- Metadata extraction and storage
- Thumbnail generation
- Image optimization for web delivery

**Annotation Service (TypeScript/Node.js)**
- Real-time annotation creation/editing
- Coordinate-based annotation positioning
- Version control for annotation changes
- WebSocket connections for live collaboration

**Comment/Thread Service (Ruby on Rails)**
- Threaded discussion management
- Notification system
- Message persistence and retrieval
- Search functionality

### 3.2 Database Design

**Entity Relationship Diagram:**

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Users    │     │   Photos    │     │ Annotations │
├─────────────┤     ├─────────────┤     ├─────────────┤
│ id (PK)     │────▶│ id (PK)     │────▶│ id (PK)     │
│ email       │     │ user_id(FK) │     │ photo_id(FK)│
│ name        │     │ filename    │     │ user_id(FK) │
│ role        │     │ file_path   │     │ x_coord     │
│ company_id  │     │ file_size   │     │ y_coord     │
│ created_at  │     │ mime_type   │     │ radius      │
│ updated_at  │     │ created_at  │     │ content     │
└─────────────┘     │ updated_at  │     │ created_at  │
                    └─────────────┘     └─────┬───────┘
                                             │
┌─────────────┐     ┌─────────────┐         │
│   Threads   │     │  Messages   │         │
├─────────────┤     ├─────────────┤         │
│ id (PK)     │────▶│ id (PK)     │         │
│annotation_id│◀────┘ thread_id(FK)        │
│ title       │     │ user_id(FK) │         │
│ status      │     │ content     │         │
│ created_at  │     │ created_at  │         │
│ updated_at  │     │ updated_at  │─────────┘
└─────────────┘     └─────────────┘
```

**Key Tables:**
- **users**: User management with role-based permissions
- **photos**: Photo metadata and storage references
- **annotations**: Coordinate-based annotations with content
- **threads**: Discussion threads linked to annotations
- **messages**: Individual messages within threads
- **companies**: Multi-tenancy support for different manufacturing companies

---

## 4. Technical Decisions & Trade-offs

### 4.1 Architecture Pattern: Microservices vs Monolith

**Decision**: Hybrid approach - Rails monolith with TypeScript microservices for real-time features

**Rationale:**
- Leverages CADDi's existing Rails expertise for rapid development
- TypeScript microservices handle WebSocket connections and real-time annotation updates
- Reduces complexity while enabling scalability for specific components

**Trade-offs:**
- ✅ Leverages existing CADDi Rails expertise for rapid development
- ✅ Microservices enable independent scaling of performance-critical components
- ✅ Allows different teams to work in parallel on different services
- ❌ Increased deployment and operational complexity
- ❌ Data consistency challenges across service boundaries
- ❌ Requires more sophisticated monitoring and debugging

### 4.2 Database Strategy

**Decision**: Single PostgreSQL database with service-specific schemas

**Rationale:**
- Maintains data consistency and simplifies transactions
- Cloud SQL provides managed backups, scaling, and maintenance
- Familiar technology for CADDi's engineering team

**Trade-offs:**
- ✅ Strong consistency guarantees
- ✅ Simplified data management
- ❌ Potential single point of failure
- ❌ May require sharding as scale increases

### 4.3 File Storage Strategy

**Decision**: Google Cloud Storage with CDN integration

**Rationale:**
- Cost-effective for large image files
- Built-in CDN for global performance
- Integrates seamlessly with GCP ecosystem

**Trade-offs:**
- ✅ Scalable and cost-effective storage
- ✅ Global content delivery
- ❌ Vendor lock-in to GCP
- ❌ Additional latency for initial uploads

### 4.4 Real-time Communication

**Decision**: WebSocket connections with Redis pub/sub

**Rationale:**
- Enables real-time annotation collaboration
- Redis provides fast message distribution
- Scales horizontally with multiple Redis instances

**Trade-offs:**
- ✅ True real-time experience
- ✅ Scalable message distribution
- ❌ Increased infrastructure complexity
- ❌ Connection state management challenges

---

## 5. Non-Functional Requirements

### 5.1 Security

**Authentication & Authorization:**
- Integration with CADDi's existing SSO system
- Role-based access control (RBAC) with three primary roles
- JWT tokens with 2-hour expiration and refresh mechanism
- API rate limiting: 1000 requests/hour per user

**Data Protection:**
- TLS 1.3 for all client-server communication
- AES-256 encryption for files at rest in GCS
- PII data encryption in database using Rails built-in encryption
- Regular security audits and penetration testing

**Compliance:**
- GDPR compliance for EU users
- SOC 2 Type II compliance for enterprise customers
- Data retention policies: 7 years for manufacturing quality records

### 5.2 Performance

**Response Time Targets:**
- API endpoints: < 200ms (95th percentile)
- Image upload: < 5 seconds for 10MB files
- Real-time annotation updates: < 100ms latency

**Scalability Targets:**
- 10,000 concurrent users
- 1 million photos stored per month
- 100,000 annotations created per day

**Optimization Strategies:**
- Image compression and multiple format generation (WebP, JPEG)
- Redis caching for frequently accessed data
- Database connection pooling
- CDN for static assets and images

### 5.3 DevOps & Monitoring

**Deployment Strategy:**
- Blue-green deployment using Kubernetes
- Feature flags for gradual rollouts
- Automated rollback on health check failures

**Monitoring Stack:**
- **Application Monitoring**: Datadog APM for distributed tracing
- **Infrastructure Monitoring**: Prometheus + Grafana
- **Error Tracking**: Sentry for exception monitoring
- **Uptime Monitoring**: Pingdom for external health checks

**Key Metrics:**
- API response times and error rates
- Database query performance
- File upload success rates
- WebSocket connection stability
- User engagement metrics

### 5.4 Testing Strategy

**Automated Testing:**
- Unit tests: 90% code coverage requirement
- Integration tests: API contract testing with Pact
- End-to-end tests: Cypress for critical user journeys
- Load testing: K6 scripts for performance validation

**Quality Gates:**
- All tests must pass before deployment
- Security scanning with Snyk
- Code quality checks with SonarQube
- Performance regression testing

---

## 6. Implementation Strategy & Resource Estimation

### 6.1 Development Approach: Value-Driven Incremental Delivery

**Core Philosophy**: Prioritize features that demonstrate immediate user value while building foundational architecture for long-term scalability.

### 6.2 Phased Development Plan

**Phase 1: Core MVP (10-12 weeks)**
- User authentication and role management
- Basic photo upload and viewing
- Simple circle annotations with basic commenting
- Essential security and compliance features

**Phase 2: Collaboration Features (6-8 weeks)**
- Real-time annotation updates
- Threaded discussions
- Notification system
- Mobile-responsive interface

**Phase 3: Enterprise Enhancement (8-10 weeks)**
- Advanced annotation tools (shapes, text, arrows)
- Search and filtering capabilities
- Analytics dashboard for managers
- API for third-party integrations

### 6.3 Resource Requirements Analysis

**Minimum Viable Team (Phase 1):**
- 1 Technical Lead/Architect
- 2 Frontend Engineers (React/TypeScript expertise)
- 2 Backend Engineers (Rails + Node.js experience)
- 1 DevOps Engineer (GCP/Kubernetes)
- 1 QA Engineer
- **Total: 7 engineers**

**Optimal Team for Parallel Development:**
- 1 Technical Lead/Architect
- 3 Frontend Engineers (Web + Mobile)
- 3 Backend Engineers (2 Rails, 1 Node.js specialist)
- 2 DevOps Engineers (Infrastructure + Security)
- 2 QA Engineers (Manual + Automation)
- 1 UX Designer (Manufacturing workflow expertise)
- **Total: 12 engineers + 1 designer**

**Specialized Skills Needed:**
- **Critical**: Rails expertise (existing CADDi strength)
- **Critical**: TypeScript/React experience
- **Important**: WebSocket/real-time systems knowledge
- **Important**: Image processing and optimization
- **Nice-to-have**: Manufacturing domain knowledge

---

## 7. Risk Assessment & Mitigation

**Technical Risks:**
- **Real-time sync complexity**: Mitigate with proven WebSocket libraries and Redis
- **Image storage costs**: Implement automatic compression and lifecycle policies
- **Database performance**: Plan for read replicas and query optimization

**Business Risks:**
- **User adoption**: Implement comprehensive user testing and feedback loops
- **Scalability demands**: Design for horizontal scaling from day one
- **Security concerns**: Engage security consultants for architecture review

**Operational Risks:**
- **Team knowledge gaps**: Provide TypeScript training for Rails developers
- **Infrastructure complexity**: Use managed services to reduce operational burden
- **Deployment issues**: Implement comprehensive CI/CD with automated testing

---

## 8. Conclusion

This architecture leverages CADDi's existing technical expertise while introducing modern technologies for real-time collaboration. The hybrid approach balances rapid development with scalability requirements, positioning the platform for both quick market entry and long-term growth.

The design prioritizes user experience through real-time features while maintaining enterprise-grade security and compliance. By utilizing managed cloud services and proven technologies, we minimize operational overhead and focus engineering resources on core business value.

**Next Steps:**
1. Technical spike for WebSocket integration patterns (1 week)
2. Database schema validation with sample data (1 week)
3. Infrastructure setup and CI/CD pipeline (2 weeks)
4. Begin MVP development with user authentication (Week 4)

This architecture provides a solid foundation for delivering a valuable manufacturing communication tool while maintaining alignment with CADDi's technical culture and operational practices.