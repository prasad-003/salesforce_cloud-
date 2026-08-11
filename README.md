<img width="1277" height="492" alt="image" src="https://github.com/user-attachments/assets/0d05c739-ce17-4047-b710-050f69bb963f" /> 

# Salesforce Apex Mailing Service

A lightweight, robust, and central utility service for managing outbound email transactions in Salesforce. This service provides structured methods for handling standard custom emails as well as Salesforce Email Templates, complete with partial success routing configurations.

## Features

- **Simple Email Broker**: Dispatches plain-text or HTML formats directly to collections of raw email strings.
- **Template Email Broker**: Drives automated delivery using existing Salesforce Email Templates, Target Object IDs (`Contact`, `Lead`, `User`), and Related Record IDs (`Account`, `Opportunity`, etc.).
- **Partial Success Routing**: Uses `Messaging.sendEmail(emails, false)` so that operational validation failures on a single bad email address will not roll back an entire transaction block.
- **Silent Exception & Error Handling**: Inspects response arrays and captures native Salesforce status codes safely in execution debug logs.

## Installation

Deploy the `MailingService.cls` file to your Salesforce Org via SFDX, VS Code, or the Salesforce Developer Console.

### MailingService Class

```java
public with sharing class MailingService {

    public static void sendSimpleEmail(List<String> toAddresses, String subject, String body, Boolean isHtml) {
        if (toAddresses == null || toAddresses.isEmpty()) {
            System.debug('MailingService Error: No recipients specified.');
            return;
        }

        Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
        mail.setToAddresses(toAddresses);
        mail.setSubject(subject);

        if (isHtml == true) {
            mail.setHtmlBody(body);
        } else {
            mail.setPlainTextBody(body);
        }

        executeSend(new List<Messaging.SingleEmailMessage>{ mail });
    }

    public static void sendTemplateEmail(Id targetObjectId, Id templateId, Id whatId) {
        if (targetObjectId == null || templateId == null) {
            System.debug('MailingService Error: targetObjectId and templateId are required for template execution.');
            return;
        }

        Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
        mail.setTargetObjectId(targetObjectId);
        mail.setTemplateId(templateId);
       
        if (whatId != null) {
            mail.setWhatId(whatId);
        }

        executeSend(new List<Messaging.SingleEmailMessage>{ mail });
    }

    private static void executeSend(List<Messaging.SingleEmailMessage> emails) {
        if (emails == null || emails.isEmpty()) {
            return;
        }

        try {
            List<Messaging.SendEmailResult> results = Messaging.sendEmail(emails, false);
            inspectResults(results);
        } catch (Exception ex) {
            System.debug('MailingService critical error during send transaction: ' + ex.getMessage());
        }
    }

    private static void inspectResults(List<Messaging.SendEmailResult> results) {
        for (Messaging.SendEmailResult result : results) {
            if (result.isSuccess()) {
                System.debug('MailingService: Email dispatched successfully.');
            } else {
                for (Messaging.SendEmailError error : result.getErrors()) {
                    System.debug('MailingService Error: ' + error.getStatusCode() + ' - ' + error.getMessage());
                }
            }
        }
    }
}
```

## Usage Examples

You can run these code snippets inside Apex Triggers, Controllers, or from the **Execute Anonymous Window** in the Developer Console.

### 1. Send Simple HTML Email
```java
List<String> recipients = new List<String>{'user@example.com'};
String mailSubject = 'Welcome to the Platform!';
String mailBody = '<h1>Hello!</h1><p>Your registration process is fully completed.</p>';

MailingService.sendSimpleEmail(recipients, mailSubject, mailBody, true);
```

### 2. Send via Email Template
```java
Contact targetCon = [SELECT Id FROM Contact LIMIT 1];
EmailTemplate activeTemplate = [SELECT Id FROM EmailTemplate WHERE DeveloperName = 'Welcome_Template' LIMIT 1];

MailingService.sendTemplateEmail(targetCon.Id, activeTemplate.Id, null);
```

## Environment Troubleshooting

If Apex executes successfully but no email drops into the recipient's inbox, verify the following configuration items:

1. **Deliverability Setting**: In Salesforce Setup, navigate to **Deliverability** and verify that the **Access Level** is explicitly set to **All email** (Sandboxes default to *No email*).
2. **Invalid Email Suffixes**: Verify that the executing user's email or target test records do not contain the automated `.invalid` sandbox suffix.
3. **Email Logs**: Use Salesforce **Email Logs** in Setup to track if the delivery status is marked as **D** (Delivered), which means the message left Salesforce but is trapped in a corporate spam filter.

