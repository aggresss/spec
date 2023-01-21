# HTTP/3

> 原文 [https://www.rfc-editor.org/rfc/rfc9114](https://www.rfc-editor.org/rfc/rfc9114)

## 摘要

## 1. Introduction

### 1.1. Prior Versions of HTTP

### 1.2. Delegation to QUIC

## 2. HTTP/3 Protocol Overview

### 2.1. Document Organization

### 2.2. Conventions and Terminology

## 3. Connection Setup and Management

### 3.1. Discovering an HTTP/3 Endpoint

#### 3.1.1. HTTP Alternative Services

#### 3.1.2. Oter Schemes

### 3.2. Connection Establishment

### 3.3. Connection Reuse

## 4. Expressing HTTP Semantics in HTTP/3

### 4.1. HTTP Message Framing

#### 4.1.1. Request Cancellation and Rejection

#### 4.1.2. Malformed Requests and Responses

### 4.2. HTTP Fields

#### 4.2.1. Field Compression

#### 4.2.2. Header Size Constraints

### 4.3. HTTP Control Data

#### 4.3.1. Request Pseudo-Header Fields

#### 4.3.2. Response Pseudo-Header Fields

### 4.4. The CONNECT Method

### 4.5. HTTP Upgrade

### 4.6. Server Push

## 5. Connection Closure

### 5.1. Idle Connections

### 5.2. Connection Shutdown

### 5.3. Immediate Application Closure

### 5.4. Transport Closure

## 6. Stream Mapping and Usage

### 6.1. Bidirectional Streams

### 6.2. Unidirectional Streams

#### 6.2.1. Control Streams

#### 6.2.2. Push Streams

#### 6.2.3. Reserved Stream Types

## 7. HTTP Framing Layer

### 7.1. Frame Layout

### 7.2. Frame Definitions

### 7.2.1. DATA

### 7.2.2. HEADERS

### 7.2.3. CANCEL_PUSH

### 7.2.4. SETTINGS

### 7.2.5. PUSH_PROMISE

### 7.2.6. GOAWAY

### 7.2.7. MAX_PUSH_ID

### 7.2.8. Reserved Frame Types

## 8. Error Handling

### 8.1. HTTP/3 Error Codes

## 9. Extensions to HTTP/3

