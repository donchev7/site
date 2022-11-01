---
date: 2022-10-31
title: 'AWS Lambda invoker identification'
slug: aws-lambda-invoker-identification
toc: false
tags:
  - Cloud
  - AWS
---

## Introduction

I am building a CLI tool that will invoke AWS Lambda functions. The tool will be used by developers to create AWS resources. You may ask why not give the developers access to AWS CLI? 

Even if I did give them AWS CLI access I cannot gurantee that the developers will adhere to the naming conventions and tagging conventions.

Another added benefit of creating a CLI tool is that developers will get the resources pre-configured with the "right" settings. E.g.

```bash
# configures rds with correct name, tags, small instance, no backups etc...
$ dev-tool rds create --test

# same but with larger instance, backups etc...
$ dev-tool rds create --prod
```

## Problem

In AWS I have enabled federated AWS access. Developers assume a role and get CLI / Console **read** access to the AWS account.
But what if the developer needs to creat secrets, RDS instances, provision state machines etc...?

Thats where the CLI tool comes in. The CLI tool invokes AWS Lambda functions that creates the resources.
But how do you know who invoked the Lambda function? How do you know if the developer is allowed to create production resources?

## Solution

In the CLI I presign the get-caller-identity call. The presigned URL is then passed to the Lambda functions. I created a lambda middleware that calls the presigned URL and gets the caller identity. Does authentication and authorization and if everything is OK it passes the user information via the context to the Lambda functions.

The solution is similar to how EKS IAM works. It is actually a very simple solution. But I could not find any documentation on how to do it. So I thought I would share it.

```go
package auth

import (
	"context"
	"encoding/xml"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"regexp"
	"strings"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/sts"
)


type getCallerIdentityResponse struct {
	XMLName                 xml.Name `xml:"GetCallerIdentityResponse"`
	Text                    string   `xml:",chardata"`
	Xmlns                   string   `xml:"xmlns,attr"`
	GetCallerIdentityResult struct {
		Text    string `xml:",chardata"`
		Arn     string `xml:"Arn"`
		UserId  string `xml:"UserId"`
		Account string `xml:"Account"`
	} `xml:"GetCallerIdentityResult"`
	ResponseMetadata struct {
		Text      string `xml:",chardata"`
		RequestId string `xml:"RequestId"`
	} `xml:"ResponseMetadata"`
}

type IAMService struct {
	aws *sts.Client
	cfg aws.Config
}

func NewIAMService(cfg aws.Config) IAMService {
	return IAMService{
		aws: sts.NewFromConfig(cfg),
		cfg: cfg,
	}
}

func (iam IAMService) GetCallerIdentity(ctx context.Context) (data.Idenity, error) {
	sign := sts.NewPresignClient(iam.aws)
	identity, err := sign.PresignGetCallerIdentity(ctx, &sts.GetCallerIdentityInput{})
	if err != nil {
		return data.Idenity{}, fmt.Errorf("could not get caller identity: %w", err)
	}

	url, err := url.Parse(identity.URL)
	if err != nil {
		return data.Idenity{}, fmt.Errorf("could not parse url: %w", err)
	}

        // only send the query params
	return data.Idenity{
		Authentication: url.RawQuery,
	}, nil
}

func (iam IAMService) VerifyIdentity(ctx context.Context, id data.Idenity) (data.Tenant, error) {
        // create the URL yourself. Do not trust the URL from the client
	authURL := fmt.Sprintf("https://sts.%s.amazonaws.com/?%s", iam.cfg.Region, id.Authentication)
	url, err := url.Parse(authURL)
	if err != nil {
		return fmt.Errorf("could not parse url: %w", err)
	}

	httpReq := &http.Request{
		Method: "GET",
		URL:    url,
	}

	response, err := http.DefaultClient.Do(httpReq)
	if err != nil {
		return fmt.Errorf("could not send request: %w", err)
	}

	body, err := io.ReadAll(response.Body)
	if err != nil {
		return fmt.Errorf("could not read response body: %w", err)
	}

	resp := getCallerIdentityResponse{}
	err = xml.Unmarshal(body, &resp)
	if err != nil {
		return fmt.Errorf("could not unmarshal response body: %w", err)
	}
  
        // do authentication and authorization here
        // the resp.GetCallerIdentityResult object has accountID, userID, the role assumed etc...

	return nil
}
```

The `GetCallerIdentity` function is called by the CLI tool. The `VerifyIdentity` function is called by the Lambda middleware.

### IMPORTANT

The `PresignGetCallerIdentity` from the AWS STS SDK will return an URL. Do not trust the URL from the client. Create the URL yourself. The client can manipulate the URL and send you a URL that points to a different website. 

Also do your own authentication / authorization checks in the `VerifyIdentity` function. You have access to the accountID, userID, the role assumed etc... You can do whatever you want with that information.

## Conclusion

I hope this helps someone. I could not find any documentation on how to do this. So I thought I would share it. As always If you have any questions or comments you can reach me on twitter [@bobby_donchev](https://twitter.com/bobby_donchev).
