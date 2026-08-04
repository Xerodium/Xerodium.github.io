+++ 
title = "Secure Code Review On A Password Change API in Dotnet"
date = "2026-08-03T00:00:00+13:00"
draft = false
+++

After being in or around the .NET ecosystem for a while, I have seen faulty authentication mechanisms from insecure password storage, MFA bypass, poor session management, and insecure password change functionality. In this article, I wanted to give a raw and honest write up of a secure code review challenge for a password change API in .NET. This is being done completely blind as I write it so I can go over my thought process.

The challenge itself is called `Bad password.cs` and can be found at [https://play.secdim.com/game/c/challenge/bad-passwordcs](https://play.secdim.com/game/c/challenge/bad-passwordcs). The description for the challenge outlines the expectations and testing harness that the platform provides.
> A password or a memorized secret must possess a variety of security features. In addition to preventing users from selecting weak passwords, the program should implement crucial security checks to make it harder for attackers to exploit weak authentication.

### Project Setup and Overview
So starting off, we have a 30 minute time limit to complete the challenge. However as I am writing this article, I won't constrain myself to that time limit. Instead I will write down the process and decisions I make while attempting the challenge.
- Opening a terminal and setting my code directory to clone the repository into.
- Creating an ED25519 SSH key pair and adding the public key to my Secdim account through the command `ssh-keygen -t ed25519 -f C:\Users\MaxFrancis\.ssh\id_ed25519`
- Testing the SSH connection to Secdim through the command `ssh -T`
![](https://raw.githubusercontent.com/Xerodium/Xerodium.github.io/main/images/test_ssh.png)
- Cloning the repository through the command `git clone ssh://git@game.secdim.com/xerodium/password.cs.git`

At this point, you'll hopefully have everything set up and ready to go. Opening up the terminal, I like to look at the folder structure expanded out to get a gauge of the scale and what the makeup of the project looks like.
![](https://raw.githubusercontent.com/Xerodium/Xerodium.github.io/main/images/layout_code.png)
Before we continue, the readme available is an instant go to. It tells you the purpose, setup commands, testing, contributing guidelines, and very much is the rules you read before diving in. If you are unfamiliar with the repository layout, read the README first.

Now with this you can see this is clearly a C# application with an associated Dockerfile and Makefile for containerized local deployment and testing. On the note of testing, within the `/test` folder there are unit and security tests the platform will run against our changes to validate behaviour.
![](https://raw.githubusercontent.com/Xerodium/Xerodium.github.io/main/images/generate_password_hash.png)
The last thing to do is to look at the C# project file (Viewable via the `.csproj` extension file) for both the program and tests to figure out some basic information. Looking at the following, the key observations are:
- Both the test and the program projects are running off the `net6.0` target framework
- The test project is utilising Moq for mocking and NUnit for the tests themselves

With that all out of the way, now comes understanding the functionality and the tests.
### Understanding the API and Test Win Conditions
I always like to start at the controller level where it's available as this most likely has the endpoints I'll be working with. So let's start by dumping out the code here:
``` dotnet
using Microsoft.AspNetCore.Mvc;
using System;
using System.Security.Cryptography;
using System.Text;

//  NOTE: You can import from existing packages
//  but no new package can be installed.

namespace program.Controllers
{
    public class ChangePasswordRequest
    {
        public string? Email { get; set; }
        public string? Password { get; set; }
        public string? NewPassword { get; set; }
    }

    [ApiController]
    public class MainController : ControllerBase
    {
        FakeDb db = FakeDb.Instance;
        private readonly HttpClient applicationHttpClient;

        public MainController(HttpClient httpClient)
        {
            applicationHttpClient = httpClient;
        }

        [HttpGet("/")]
        public IActionResult Get()
        {
            return Ok("curl -X POST http://localhost:8080/change-password" +
                          " -H 'Content-Type: application/json'" +
                          " -d '{\"Email\":\"alice@secdim.com\", \"Password\":\"py]XxE9K4z.w_[vb\", \"NewPassword\": \"RR3@=%k7;`RYe8.X\"}'");
        }

        [HttpPost("/change-password")]
        public IActionResult Post([FromBody] ChangePasswordRequest request)
        {
            if (request.Email == null || request.Password == null || request.NewPassword == null)
            {
                return StatusCode(422, "Data is unprocessable");
            }

            User? dbUser = db.GetUser(request.Email);
            if (dbUser == null)
            {
                return StatusCode(404, "User not found");
            }

            string passwordHash = db.GeneratePasswordHash(request.Email, request.Password);
            if (db.VerifyPassword(passwordHash, dbUser.PWHash))
            {
                db.UpdatePassword(request.Email, request.NewPassword);
                return StatusCode(200, $"User {request.Email} password successfully changed");
            }
            else
            {
                return StatusCode(403, "Old password is wrong");
            }
        }
    }
}
```
Now let's point out the obvious, for the `ChangePasswordRequest` class we take in an email, password, and new password all as strings. When we move down to the `[ApiController]` class level attribute, the route definitions and error handling show the expected API surface.
- HTTP GET / = This is a GET request that returns a message showcasing how to call the API on localhost 8080
- HTTP POST /change-password = This performs a change password request and either returns
  - a 200 with a success message response with users email in the body
  - a 404 when a user is not found
  - a 422 when one of the aforementioned values aren't provided
  - a 403 when the provided password is wrong

Now with what we currently have, I'd feel comfortable enough to run the tests and see what gets spat out. We'll do this with `make build`, `make test`, and `make securitytest` (you'd know these commands exist in the Makefile).
![](https://raw.githubusercontent.com/Xerodium/Xerodium.github.io/main/images/security_test.png)
Oh noes! We haz bugz! This is expected due to this being the start of the challenge. Counting across the board we have 7 security tests and 5 usability tests. Not to discount the usability but as a starting point I focus on the security tests.

**test_change_password_cannot_set_weak_pw_per_nist_800_63B_return_403** which checks we have NIST password guidelines for our passwords being 16 characters, one number, one uppercase

Now the other 6 only contain `Assert.AreEqual(true, false);` which will always be false. This tells us that either we shouldn't worry about them right now, or the test logic is in the Origin server where the platform validates additional constraints.
### First fix, setting password requirements
So immediately my instinct is to logically map out the flow in where to check the password complexity. This may seem trivial and to a degree it is in an application of this scale, however with time constraints we should be deliberate about where to add checks.
```
-----------------     -----------     --------     ----------------     ---------     ---------
request pw change --> check nulls --> get user --> generate pw hash --> verify pw --> update pw 
-----------------     -----------     --------     ----------------     ---------     ---------
```
So in effect we need to inject in where as part of the flow we'd need to include the **check new pw complexity**. Due to the compute of the feature itself and the unnecessity of needing to know the internal implementation details for the test harness, we can call into a helper which validates complexity and breached passwords.
```
-----------------     -----------     -----------------------    --------     ----------------     ---------     ---------
request pw change --> check nulls --> check new pw complexity -> get user --> generate pw hash --> verify pw --> update pw 
-----------------     -----------     -----------------------    --------     ----------------     ---------     ---------
```
Now once I put that code through to both security and usability tests, I was unfortunately hit with failing usability tests even though in my head I had not written any code that should have broken them.
```
...

  Failed test_change_password_invalid_user_should_return_http_not_found [82 ms]

  Error Message:

     Expected: 404

  But was:  422

  Failed test_change_password_valid_user_invalid_old_password_should_return_http_forbidden [1 ms]

  Error Message:
     Expected: 403
  But was:  422
...

Failed!  - Failed:     2, Passed:     3, Skipped:     0, Total:     5, 
```

Now I looked into this and realized that the error code was correct, but what was happening is the usability tests were inappropriately written for this challenge in a way that weak password checks interfered with the expected responses.
![](https://raw.githubusercontent.com/Xerodium/Xerodium.github.io/main/images/munger.jpg)

``` dotnet
        [HttpPost("/change-password")]
        public IActionResult Post([FromBody] ChangePasswordRequest request)
        {
            if (request.Email == null || request.Password == null || request.NewPassword == null)
            {
                return StatusCode(422, "Data is unprocessable");
            }

            User? dbUser = db.GetUser(request.Email);
            if (dbUser == null)
            {
                return StatusCode(404, "User not found");
            }

            string passwordHash = db.GeneratePasswordHash(request.Email, request.Password);
            if (db.VerifyPassword(passwordHash, dbUser.PWHash))
            {
                if (request.Password == request.NewPassword)
                {
                    return StatusCode(403, "New password cannot be the same as the old password");
                }
                if (!db.VerifyNewPasswordMeetsComplexity(request.NewPassword))
                {
                    return StatusCode(403, "New password does not meet complexity requirements. Please ensure 16 characters, upper case, lowercase, numbers and special characters");
                }

                db.UpdatePassword(request.Email, request.NewPassword);
                return StatusCode(200, $"User {request.Email} password successfully changed");
            }
            else
            {
                return StatusCode(403, "Old password is wrong");
            }
        }
```
Overall, we stick that VerifyNewPasswordMeetsComplexity to return a 403 from within 

`test_change_password_oldpw_newpw_cannot_be_same_403`

``` dotnet
            if (db.VerifyPassword(passwordHash, dbUser.PWHash))
            {
                if (request.Password == request.NewPassword)
                {
                    return StatusCode(403, "New password cannot be the same as the old password");
                }
                if (!db.VerifyNewPasswordMeetsComplexity(request.NewPassword))
                {
                    return StatusCode(403, "New password does not meet complexity requirements. Please ensure 16 characters, upper case, lowercase, numbers and special characters");
                }

                db.UpdatePassword(request.Email, request.NewPassword);
                return StatusCode(200, $"User {request.Email} password successfully changed");
            }
```
Now after this, we should commit what we currently have upstream to the origin server (i.e., the SecDim platform) so that the usability and security tests can be properly run. Usually you can use the following commands:
- `git add .` which will add the files on and deeper to where you currently are to the prepared commit
- `git commit -m "message"` set a commit with a message around what was changed
- `git push` which will push the commit set to the origin server.

One should also be cognisant that there are other commands that might need to be set up, such as configuring the username or even installing the git CLI in general. I'd recommend checking out https://git-scm.com for a reference if you need help getting started.

![](https://raw.githubusercontent.com/Xerodium/Xerodium.github.io/main/images/git.png)

Although we committed the code and all was well. We then hit another roadblock being that the password needs to be checked against a list of previously breached hashed passwords.
![](https://raw.githubusercontent.com/Xerodium/Xerodium.github.io/main/images/failed_tests2.png)

Having a look at the tests themselves, it appeared that the hashes were SHA-1'ed and the API took an endpoint for the first five characters of the hash, and then compared the rest within the response body.
``` dotnet
public class programSecurityTest
{
    ...
            .Returns((HttpRequestMessage request, CancellationToken cancellationToken) =>
            {
                if (request?.RequestUri?.ToString().Contains("https://api.pwnedpasswords.com/range/A7B5F") ?? false)
                {
                    ...
                }
                else if (request?.RequestUri?.ToString().Contains("https://api.pwnedpasswords.com/range/5EDAE") ?? false)
                {
                    ...
                }
            }
```

In order to best achieve this security test and allow the pass, we pass both the httpClient and the newPassword to check against the pwnedpasswords API like so, calling the Database function from the tests:

``` dotnet
        public bool VerifyPasswordNotInBreachDatabase(string newPassword, HttpClient client)
        {
            byte[] hashBytes = SHA1.HashData(Encoding.UTF8.GetBytes(newPassword));
            string hash = Convert.ToHexString(hashBytes);
            string prefix = hash[..5];

            string apiResponse = client.GetStringAsync($"https://api.pwnedpasswords.com/range/{prefix}").GetAwaiter().GetResult();

            return !apiResponse.Contains(hash[5..], StringComparison.OrdinalIgnoreCase);
        }
```

Now the rest of these tests included two more security tests, I won't get into the specifics around what I did or how I did it as this article is getting long, however there are two more tests that were relevant:
- `test_generate_password_must_use_sha512_1000itr` self explanatory but this involved changing the SHA-1 password hashing algorithm to SHA-512, adjusting the chunked sizes, and upping the iterations.
- `test_must_have_unique_salt_per_user` this one just involves changing the salt, recomputing the hashes, and updating the values for the test users. We didn't have to write any logic around unique salt generation, just ensure each user has a unique salt.

![](https://raw.githubusercontent.com/Xerodium/Xerodium.github.io/main/images/passed_tests.png)

Overall this would be a good start. If I had full remit of the password change of any company I've worked for before (this can be hard with responsibilities, prioritization and politics), I'd include similar checks and testing as part of a secure deployment pipeline.

Well that is all from now. I'll leave this by saying I am not an oracle of secure code review, and am downright shocked at my lack of software principles and ability to produce clean code in comparison to many people who do it for a living. Thanks for reading.
