# HTTP/1.1

> 原文 [https://www.rfc-editor.org/rfc/rfc9112](https://www.rfc-editor.org/rfc/rfc9112)

## 摘要

## 1. Introduction

## 2. Message

### 2.1. Message Format

### 2.2. Message Parsing

### 2.3. HTTP Version

## 3. Request Line

### 3.1. Method

### 3.2. Request Target

#### 3.2.1. origin-form

#### 3.2.2. absolute-form

#### 3.2.3. authority-form

#### 3.2.4. asterisk-from

### 3.3. Reconstructing the Target URI

## 4. Status Line

## 5. Field Syntax

### 5.1. Field Line Parsing

### 5.2. Obsolete Line Folding

## 6. Message Body

### 6.1. Transfer-Encoding

### 6.2. Content-Length

### 6.3. Message Body Length

## 7. Transfer Codings

### 7.1. Chunked Transfer Coding

#### 7.1.1. Chunk Extensions

#### 7.1.2. Chunked Trailer Section

#### 7.1.3. Decoding Chunked

### 7.2. Transfer Codings for Compression

### 7.3. Transfer Codings Registry

### 7.4. Negotiating Transfer Codings

## 8. Handling Incomplete Messages

## 9. Connection Management

### 9.1. Establishment

### 9.2. Associating a Response to a Request

### 9.3. Persistence

#### 9.3.1. Retrying Requests

#### 9.3.2. Pipelining

### 9.4. Concurrency

### 9.5. Failures and Timeouts

### 9.6. Tear-down

### 9.7. TLS Connection Initiation

### 9.8. TLS Connection Closure

## 10. Enclosing Messages as Data

### 10.1. Media Type message/http

### 10.2. Media Type application/http
