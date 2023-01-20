# HTTP Caching

> 原文 [https://www.rfc-editor.org/rfc/rfc9111](https://www.rfc-editor.org/rfc/rfc9111)

## 摘要

## 1. 介绍

## 2. Overview of Cache Operation

## 3. Storing Responses in Caches

### 3.1. Storing Header and Trailer Fields

### 3.2. Updating Stored Header Fields

### 3.3. Storing Incomplete Responses

### 3.4. Combining Partial Content

### 3.5. Storing Responses from Caches

## 4. Constructing Responses from Caches

### 4.1. Calculating Cache Keys with the Vary Header Field

### 4.2. Freshness

#### 4.2.1. Calculating Freshness Lifetime

#### 4.2.2. Calculating Heuristic Freshness

#### 4.2.3. Calculating Age

#### 4.2.4. Serving Stale Responses

### 4.3. Validation

#### 4.3.1. Sending a Validating Request

#### 4.3.2. Handling a Received Validation Request

#### 4.3.3. Handling a Validation Response

#### 4.3.4. Freshening Stored Responseds upon Validation

#### 4.3.5. Freshening Responses with HEAD

### 4.4. Invalidating Stored Responses

## 5. Field Deinitions

### 5.1. Age

### 5.2. Cache-Control

#### 5.2.1. Request Directives

##### 5.2.1.1. max-age

##### 5.2.1.2. max-stale

##### 5.2.1.3. min-fresh

##### 5.2.1.4. no-cache

##### 5.2.1.5. no-store

##### 5.2.1.6. no-transform

##### 5.2.1.7. only-if-cached

#### 5.2.2. Response Directives

##### 5.2.2.1. max-age

##### 5.2.2.2. must-revalidate

##### 5.2.2.3. must-understand

##### 5.2.2.4. no-cache

##### 5.2.2.5. no-store

##### 5.2.2.6. no-transform

##### 5.2.2.7. private

##### 5.2.2.8. proxy-revalidate

##### 5.2.2.9. public

##### 5.2.2.10. s-maxage

#### 5.2.3. Extension Directives

#### 5.2.4. Cache Directive Registry

### 5.3. Expires

### 5.4. Pragma

### 5.5. Warning

## 6. Relationship to Applications and Other Caches
