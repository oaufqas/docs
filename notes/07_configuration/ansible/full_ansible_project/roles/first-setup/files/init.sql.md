```sql
CREATE DATABASE IF NOT EXISTS `kyc_rent`;
USE `kyc_rent`;

DROP TABLE IF EXISTS `users`;

CREATE TABLE `users` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `email` VARCHAR(255) NOT NULL UNIQUE,
    `password` VARCHAR(255) NOT NULL,
    `userName` VARCHAR(255) DEFAULT 'user',
    `isActivated` TINYINT(1) DEFAULT false,
    `activationLink` VARCHAR(255),
    `role` ENUM('user', 'admin') NOT NULL DEFAULT 'user',
    `twoFactorCode` VARCHAR(6) DEFAULT NULL,
    `twoFactorExpires` DATETIME DEFAULT NULL,
    `balance` DECIMAL(10,2) DEFAULT 0,
    `status` ENUM('unblocked', 'blocked') DEFAULT 'unblocked',
    `createdAt` DATETIME NOT NULL,
    `updatedAt` DATETIME NOT NULL
);

DROP TABLE IF EXISTS `userTokens`;

CREATE TABLE `userTokens` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `userId` INT NOT NULL,
    `refreshToken` VARCHAR(512) NOT NULL,
    `createdAt` DATETIME NOT NULL,
    `updatedAt` DATETIME NOT NULL,
    FOREIGN KEY (`userId`) REFERENCES `users` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
);

DROP TABLE IF EXISTS `accounts`;

CREATE TABLE `accounts` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `account_number` INTEGER NOT NULL UNIQUE,
    `title` VARCHAR(255) NOT NULL,
    `description` TEXT NOT NULL,
    `characters` VARCHAR(255) NOT NULL DEFAULT '{"bape":false,"crewUniform":false,"more300mif":false}',
    `price` INTEGER NOT NULL DEFAULT 200,
    `status` ENUM('rented', 'free', 'unavailable', 'deleted') NOT NULL DEFAULT 'unavailable',
    `rentExpiresAt` DATETIME,
    `img` VARCHAR(255) NOT NULL,
    `video` VARCHAR(255),
    `createdAt` DATETIME NOT NULL,
    `updatedAt` DATETIME NOT NULL
);

DROP TABLE IF EXISTS `orders`;

CREATE TABLE `orders` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `userId` INT,
    `accountId` INT,
    `transactionId` INT,
    `amount` DECIMAL(10,2) NOT NULL,
    `status` ENUM('pending', 'paid', 'verified', 'active', 'completed', 'cancelled') NOT NULL,
    `rentPeriod` INTEGER NOT NULL,
    `startsAt` DATETIME,
    `expiresAt` DATETIME,
    `check` VARCHAR(255),
    `verificationPlatform` VARCHAR(255) NOT NULL,
    `userNameInPlatform` VARCHAR(255) NOT NULL,
    `canReview` TINYINT(1) DEFAULT false,
    `hasReview` TINYINT(1) DEFAULT false,
    `canSendMail` TINYINT(1) DEFAULT true,
    `paymentMethod` ENUM('balance', 'bank_transfer', 'crypto') NOT NULL,
    `createdAt` DATETIME NOT NULL,
    `updatedAt` DATETIME NOT NULL,
    FOREIGN KEY (`userId`) REFERENCES `users` (`id`) ON DELETE SET NULL ON UPDATE CASCADE,
    FOREIGN KEY (`accountId`) REFERENCES `accounts` (`id`) ON DELETE SET NULL ON UPDATE CASCADE
);

DROP TABLE IF EXISTS `transactions`;

CREATE TABLE `transactions` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `userId` INT,
    `orderId` INT,
    `type` ENUM('deposit', 'payment') NOT NULL,
    `amount` DECIMAL(10,2) NOT NULL,
    `status` ENUM('pending', 'completed', 'cancelled') DEFAULT 'pending',
    `method` ENUM('balance', 'bank_transfer', 'crypto') NOT NULL,
    `description` VARCHAR(255),
    `metadata` JSON,
    `check` VARCHAR(255),
    `adminNotes` VARCHAR(255),
    `createdAt` DATETIME NOT NULL,
    `updatedAt` DATETIME NOT NULL,
    FOREIGN KEY (`userId`) REFERENCES `users` (`id`) ON DELETE SET NULL ON UPDATE CASCADE,
    FOREIGN KEY (`orderId`) REFERENCES `orders` (`id`)
);

DROP TABLE IF EXISTS `reviews`;

CREATE TABLE `reviews` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `orderId` INT NOT NULL,
    `userId` INT,
    `accountId` INT,
    `ratingValue` INTEGER NOT NULL,
    `comment` TEXT,
    `status` ENUM('pending', 'approved', 'rejected') DEFAULT 'pending',
    `createdAt` DATETIME NOT NULL,
    `updatedAt` DATETIME NOT NULL,
    FOREIGN KEY (`orderId`) REFERENCES `orders` (`id`) ON DELETE CASCADE ON UPDATE CASCADE,
    FOREIGN KEY (`userId`) REFERENCES `users` (`id`) ON DELETE SET NULL ON UPDATE CASCADE,
    FOREIGN KEY (`accountId`) REFERENCES `accounts` (`id`) ON DELETE SET NULL ON UPDATE CASCADE
);

DROP TABLE IF EXISTS `paymentMethods`;

CREATE TABLE `paymentMethods` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `name` VARCHAR(255) NOT NULL UNIQUE,
    `details` VARCHAR(255) NOT NULL,
    `type` ENUM('bank_transfer', 'crypto') NOT NULL,
    `isActive` TINYINT(1) DEFAULT true,
    `createdAt` DATETIME NOT NULL,
    `updatedAt` DATETIME NOT NULL
);

ALTER TABLE `orders` ADD FOREIGN KEY (`transactionId`) REFERENCES `transactions` (`id`) ON DELETE SET NULL ON UPDATE CASCADE;

CREATE INDEX idx_users_email ON `users`(`email`);
CREATE INDEX idx_users_activationLink ON `users`(`activationLink`);
CREATE INDEX idx_users_twoFactorExpires ON `users`(`twoFactorExpires`);

CREATE INDEX idx_userTokens_userId ON `userTokens`(`userId`);
CREATE INDEX idx_userTokens_refreshToken ON `userTokens`(`refreshToken`);

CREATE INDEX idx_accounts_status ON `accounts`(`status`);
CREATE INDEX idx_accounts_rentExpiresAt ON `accounts`(`rentExpiresAt`);

CREATE INDEX idx_orders_status ON `orders`(`status`);
CREATE INDEX idx_orders_userId ON `orders`(`userId`);
CREATE INDEX idx_orders_accountId ON `orders`(`accountId`);
CREATE INDEX idx_orders_expiresAt ON `orders`(`expiresAt`);
CREATE INDEX idx_orders_canSendMail ON `orders`(`canSendMail`);

CREATE INDEX idx_transactions_userId ON `transactions`(`userId`);
CREATE INDEX idx_transactions_orderId ON `transactions`(`orderId`);
CREATE INDEX idx_transactions_status ON `transactions`(`status`);

CREATE INDEX idx_reviews_orderId ON `reviews`(`orderId`);
CREATE INDEX idx_reviews_userId ON `reviews`(`userId`);
CREATE INDEX idx_reviews_accountId ON `reviews`(`accountId`);
CREATE INDEX idx_reviews_status ON `reviews`(`status`);

CREATE INDEX idx_paymentMethods_name ON `paymentMethods`(`name`);
CREATE INDEX idx_paymentMethods_type ON `paymentMethods`(`type`);
```