# HTTP Semantics

> 原文 [https://www.rfc-editor.org/rfc/rfc9110](https://www.rfc-editor.org/rfc/rfc9110)

## 摘要

超文本传输协议 (HTTP) 是一种用于分布式、协作的超文本信息系统的无状态应用层协议。本文档描述了 HTTP 的整体架构，建立了通用术语，并从协议中所有版本共享的层面进行定义。在这个定义中包括核心协议元素、可扩展性机制以及 "http" 和 "https" 统一资源标识符 (URI) 方案。

本文档更新了RFC 3864，废止了 RFC 2818、7231、7232、7233、7235、7538、7615、7694 和 7230 的部分内容。

## 1. Introduction

### 1.1. Purpose

### 1.2. History and Evolution

### 1.3. Core Semantics

### 1.4. Specifications Obsoleted by This Document

## 2. Conformance

### 2.1. Syntax Notation

### 2.2. Requirements Notation

### 2.3. Length Requirements

### 2.4. Error Handing

### 2.5. Protocol Version

## 3. Terminology and Core Concepts

### 3.1. Resources

### 3.2. Representations

### 3.3. Connections, Clients, and Servers

### 3.4. Messages

### 3.5. User Agents

### 3.6. Origin Server

### 3.7. Intermediaries

### 3.8. Caches

### 3.9. Example Message Exchange

## 4. Identifiers in HTTP

### 4.1. URI References

### 4.2. HTTP-Related URI Schemes

#### 4.2.1. http URI Scheme

#### 4.2.2. https URI Scheme

#### 4.2.3. http(s) Normalization and Comparison

#### 4.2.4. Deprecation of userinfo in http(s) URIs

#### 4.2.5. http(s) References with Fragment Identifiers

### 4.3 Authoritative Access

#### 4.3.1. URI Origin

#### 4.3.2. http Origins

#### 4.3.3. https Origins

#### 4.3.4. https Certificate Verification

#### 4.3.5. IP-ID Reference Identity

## 5. Fields

### 5.1. Field Names

### 5.2. Field Lines and Combined Field Value

### 5.3. Field Order

### 5.4. Field Limits

### 5.5. Field Values

### 5.6. Common Rules for Defining Field Values

#### 5.6.1. Lists (#rule ABNF Extension)

##### 5.6.1.1. Sender Requirements

##### 5.6.1.2. Recipient Requirements

#### 5.6.2. Tokens

#### 5.6.3. Whitespace

#### 5.6.4. Quoted Strings

#### 5.6.5. Comments

#### 5.6.6. Parameters

#### 5.6.7. Date/Time Formats

## 6. Message Abstraction






