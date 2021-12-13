---
id: changelog
title: Changelog
sidebar_label: CHANGELOG
---
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Updated

- custom plugin lib v4.1.1

## 2.3.0 - 22-09-2021

### Changed
- [Issue 25](https://git.tools.mia-platform.eu/platform/core/flow-manager/-/issues/25): now the kafka brokers into the service configurations can be one of the following:
  - a list of strings that represent the kafka brokers
  - a string with the comma separated value brokers

## 2.2.0 - 05-05-2021

### Added

- Persistency Manager support to MongoDB

## 2.1.3 - 04-03-2021

### Updated

- custom-plugin-lib: v2.0.4

## 2.1.2 - 18-01-2020

### Added

- add custom kafka logger to follow mia-platform logs guidelines

## 2.1.1 - 20-11-2020

### Fixed

- removed consumerGroup requirement for kafka producer

## 2.1.0 - 15-10-2020

### Added

- defined new route to expose flow-manager state machine configuration

## 2.0.2 - 13-10-2020

### Updated

- custom-plugin-lib: v2.0.4

## 2.0.1 - 02-10-2020

### Updated

- custom-plugin-lib: v2.0.3

## 2.0.0 - 30-09-2020

### Added

 - concept of Business Events

### Updated

**BREAKING CHANGE**
- updated custom-plugin-lib dependency to 2.0.2. The update in breaking since it's bringing up lc39 v3.x with the newer logging format.
- updated flow history format

## 1.1.0 - 11-09-2020

### Added

 - support for REST trigger
 - flow history

## 1.0.0 - 17-07-2020

 - First import