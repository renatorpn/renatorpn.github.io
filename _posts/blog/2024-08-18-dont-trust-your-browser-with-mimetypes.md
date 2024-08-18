---
layout: post
title:  "Don't trust your browser with MIME types"
date: 2024-08-18
category: blog
tags: application-security 
---

## I used to trust my browser

If you have been in cybersecurity for some time you already know that dealing with user input sucks, but if you happen to have been in the web appsec space long enough you've come to fear file upload. I used to blindly trust the MIME type verification and stick to the recommendation that once you have have an allow-list of trusted MIME types you're good to go.

But I could have never been more wrong. Also, apparently I don't know how to read because [I just ignored what OWASP had to say](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html).

## But first - What is a MIME type?

MIME (a.k.a Multipurpose Internet Mail Extension) is used to determine the format of a file (or any stream of bytes for that matter). Whenever you see something like `application/pdf` you're staring at a MIMEtype and it is used to indicate a `type/subtype` format.
Types can be either **discrete** or **multipart**. **Discrete types** are absolute types of data, such as a single text, audio or image file. **Multipart types**, as the name implies, are types of data that are broken into pieces.

Multipart types can have their own MIME types, with each piece of data (or file) having their own type and name. Whenever we have a form with a file upload it means that we expect a `multipart/form-data` kind of media. Why is that, you might ask and it is, as W3 themselves said, `application/x-www-form-urlenconded` is kinda crap dealing with large quantities of binary data or any non-ASCII characters. Thus `multipart/form-data` was born.

[RFC 2183](https://www.ietf.org/rfc/rfc2388.txt) states:
>"multipart/form-data" contains a series of parts. Each part is expected to contain a content-disposition header [RFC 2183] where the disposition type is "form-data", and where the disposition contains an (additional) parameter of "name", where the value of that parameter is the original field name in the form.

## A Simple NextJS Application

To understand how MIME types actually work I wrote the most basic upload application with the most non sensical in-line CSS because I still don't understand how stylesheets work. I wrote it with NextJS and in this application I'll have just a simple form.

This form will take a file upload and redirect to the API endpoint. This endpoint which we'll call `/upload-mime` contains a file validation logic to check if the file is a PDF file or not. If we determine that it is in fact a PDF file, we want to save it in a path otherwise I want this application to shame the user for trying to bypass it.

That's how I went into writing `/upload-mime`:

```javascript
export async function POST(req: NextRequest) {
  const contentType = req.headers.get('content-type') || '';
  if (!contentType.includes('multipart/form-data')) {
    return NextResponse.json({ error: 'Invalid content type' }, { status: 400 });
  }

  const formData = await req.formData();
  const file = formData.get('file') as File;

  if (!file) {
    return NextResponse.json({ error: 'No file uploaded' }, { status: 400 });
  }

  const mimeType = file.type;
  if (mimeType !== 'application/pdf') {
    return NextResponse.json({ message:'File is not a valid PDF', error: 'File is not a valid PDF' }, { status: 400 });
  }

  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);
  const path = `./public/uploads/${file.name}`;

  try {
    await writeFile(path, buffer);
    return NextResponse.json({ message: 'File uploaded successfully' }, { status: 200 });
  } catch (error) {
    return NextResponse.json({ message:'File is not a valid PDF', error: 'Error saving the file' }, { status: 500 });
  }
}
```

It's simple enough - It's a `POST` function that accepts the contents of the file I upload through the form and that's why I'm checking if it's a `multipart/form-data` (we'll get there in a minute). After deciding that's a valid file upload, I'll start receiving the contents of that form as a `File` type. Once I have the file I'll check with `file.type` if this file is a PDF file with `application/file`, and if it isn't I want to let the user know that this is not the file type we're looking for.

What if I decided to upload this photo of a random FIFA 2003 PS2 game I stumbled the other day in a grimy bar restroom? The application certainly will say that this is not a PDF.

![Cursed FIFA Photo](/assets/images/cursed_FIFA_photo.jpg){: width="400" .center}

🔥 This pic goes hard feel free to screenshot it. 🔥

So if I decided to change the picture file extension from .png to .pdf we would get an error, right? Those are two completely different files, right? To the eyes of your browser, wrong.

## What happened?

It's time to break the ole reliable Burp Suite and get to proxying. Here's what a request to `/upload-mime` looks like:

```http
POST /api/upload-mime HTTP/1.1
Host: localhost:3000
Content-Length: 772485
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary1pTSS2WiPlkiFdkO
Accept-Language: pt-PT
sec-ch-ua-mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.6533.100 Safari/537.36
sec-ch-ua-platform: "Windows"
Accept: */*
Origin: http://localhost:3000
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: http://localhost:3000/
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

------WebKitFormBoundary1pTSS2WiPlkiFdkO
Content-Disposition: form-data; name="file"; filename="best_book_ever.pdf"
Content-Type: application/pdf

%PDF-1.5
%âãÏÓ
1430 0 obj<</H[755 2036]/Linearized 1/E 73914/L 772270/N 116/O 1433/T 743639>>
endobj
            
xref
1430 22
0000000016 00000 n
[...]

```

So, here are some interesting field in our request:

* `Content-Type: multipart/form-data` &rarr; The `Content-Type` is used to tell the application what type of media we're sending to the application.  This type of media is defined by what is called a **MIME Type**.
* `Content-Disposition: form-data; name="file"; filename="best_book_ever.pdf"` &rarr; Whenever there's a `multipart/form-data` being sent, the `Content-Disposition` is the header that it is going to tell us information about each field sent by the request, split in subparts. Here we're telling that we're sending the `file` field and the `filename` we're sending.
* `Content-Type: application/pdf` &rarr; For each subpart of our form, the application must also need to know what kind of media it's expecting.
* `body` &rarr; Here we have the contents of the file in its entirety. Notice the file header shows `%PDF` which converted to hex is `25504446`. Keep that in your mind. ;)

Know that we know what a normal request looks like, let's get any file and rename the extension to `.pdf`. For this test I'm using the Windows Signal installer, which is a portable executable:

```bash
$ file SignalSetup.pdf
SignalSetup.pdf: PE32 executable (GUI) Intel 80386, for MS Windows
```

Let's see what goes through our Burp proxy again:

```HTTP
POST /api/upload-mime HTTP/1.1
Host: localhost:3000
Content-Length: 132948690
sec-ch-ua: "Chromium";v="127", "Not)A;Brand";v="99"
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryqCgjZhz9LMvsHr9J
[...]
Connection: keep-alive

------WebKitFormBoundaryqCgjZhz9LMvsHr9J
Content-Disposition: form-data; name="file"; filename="SignalSetup.pdf"
Content-Type: application/pdf

MZ[...]This program cannot be run in DOS mode.[...].text[...].rdata[...]
```

Notice that the `Content-Type: application/pdf` in our request? Something's suspicious.

## You can't trust browsers

Grab your pitchforks - The culprit is your browser and JavaScript.

Let's go back to our `/upload-mime` endpoint and look at our culprit:

```javascript
const formData = await req.formData();
const file = formData.get('file') as File;
[...]
const mimeType = file.type;
if (mimeType !== 'application/pdf') {
    return NextResponse.json({ message:'File is not a valid PDF', error: 'File is not a valid PDF' }, { status: 400 });
 }
```

The `File` is a interface that allows JavaScript to access their contents. **More importantly a `File` extends from `Blob`.** Per definition a `Blob` is any byte sequence, that has a `size` and a `type`.

As you might have guessed the `type` property returns the file's MIME type, however here's the catch: **Browsers do not read the bytestream of what is being sent to them. It is always assumed by their file extension.**

[Let's see what Mozilla has to say about Blob types](https://developer.mozilla.org/en-US/docs/Web/API/Blob/type)
> Based on the current implementation, browsers won't actually read the bytestream of a file to determine its media type. It is assumed based on the file extension; a PNG image file renamed to .txt would give "_text/plain_" and not "_image/png_".

## Defining MIME ourselves with 🪄 Magic Numbers🪄

Let's look at the other endpoint I called `/upload-file-inspection`:

```javascript
const fileTypeCheck = (buffer: Buffer): Promise<string> => {
    return new Promise((resolve) => {
        const header = buffer.slice(0, 4).toString('hex').toLowerCase();
        let type = 'unknown';
        switch (header) {
            case '25504446':
                type = 'application/pdf';
                break;
        }
        resolve(type);
    });
};

export async function POST(req: NextRequest) {
    const data = await req.formData();
    const file = data.get('file') as unknown as File;

    if (!file) {
        return NextResponse.json({ message: 'No file uploaded' }, { status: 400 });
    }

    const bytes = await file.arrayBuffer();
    const buffer = Buffer.from(bytes);
    const path = `./public/uploads/${file.name}`;
    
    const fileType = await fileTypeCheck(buffer);
    if (fileType !== 'application/pdf') {
        return NextResponse.json({ message: `File is not a valid PDF, this is a ${fileType}!` }, { status: 400 });
    }

    await writeFile(path, buffer);
    console.log(`File uploaded: ${file.name}`);
    return NextResponse.json({ message: 'File uploaded' }, { status: 200 });
}
```

My approach here is different - We're not happy with how browsers infer MIME types, we create a byte stream reader. These stream of bytes from the file will go to the helper function `fileTypeCheck`. This function will look through the first 4 bytes from that buffer and convert to hexadecimal values. Remember when I said to keep this value `25504446` in mind? This value hex to text equals `%PDF`. Those are known as Magic Numbers, which can be used to identify file signatures.

Alternatively instead of writing ourselves the header verification ourselves we can also use other people's more stable, completed and tested library to do this work for us, like [`magic-bytes`]([LarsKoelpin/magic-bytes: A library for detecting file types. (github.com)](https://github.com/LarsKoelpin/magic-bytes/tree/master)) for example.

In an ideal world that would be enough - But what happens if I concatenate a completely different file into the PDF?

## File Normalization

 Let's say I want to sneak that cursed FIFA photo into my PDF:

```bash
cat best_book_ever.pdf cursed_FIFA_photo.png > cursed_book_ever.pdf
```

This will bypass our file verification because we're just validating the header from that stream of bytes. After reading a little bit about the PDF specification I discovered that the last line of any PDF document contains the `%%EOF` string.

With this information in mind we can expand upon our previous approach and add a function to normalize the PDF. This function will examine the last 5 bytes from that stream of bytes, look for  `2525454f46` (`%%EOF` in Hex) and return the first position of the stream until the position we found our `%%EOF` for a nice and fresh PDF.

```javascript
async function normalizePDF(buffer: Buffer): Promise<Buffer> {
    return new Promise((resolve, reject) => {
        const eofMarker = Buffer.from('2525454f46', 'hex');
        const eofIndex = buffer.lastIndexOf(eofMarker);
        if (eofIndex === -1) {
          return reject(new Error('Invalid PDF file: Missing %%EOF marker'));
        }
        const slicedBuffer = buffer.slice(0, eofIndex + eofMarker.length);
        resolve(slicedBuffer);
      });
}
```

In a real world scenario and in a real application this would be vastly different and more complex depending on the type and how clean you'd like your data. For example, if you have a image sharing application you'd have to write a service to perform this sanitization task if you want to avoid people using steganography techniques to put other types of content inside images.

## In Conclusion

* **If you have a service that allows file upload, users can and will try to exploit** &rarr; You have to think that people will bypass your application.
* **Do not rely on MIME type alone** &rarr; MIME type doesn't mean anything. Javascript will only look for the file extension. You'll need to look into the bytestream and detect the content type yourself.
* **File normalization is a major pain-in-the-ass but it needs to be done** &rarr;  People will also want to pull a trojan horse trick on your application and sneak different types of data. Obviously soon enough people will try to upload all sorts of not-so-nice contents into files. Pay special attention to [what OWASP has to say regarding File Content Validation]([File Upload - OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html#file-content-validation)). Seriously. I (and a lot of web applications) used to skip this section.

## References

* [MIME types](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types)
* [Introduction to HTTP Multipart](https://blog.adamchalmers.com/multipart/)
* [MIME Type Validation Sucks](https://flower.codes/2016/07/12/mime-type-validation.html)
* [Understanding multipart/form-data in HTTP protocol](https://www.sobyte.net/post/2021-12/learn-about-http-multipart-form-data/)
* [Forms in HTML documents](https://www.w3.org/TR/html401/interact/forms.html#h-17.13.4.2)
* [Understanding PDF Vulnerabilities and Shellcode Attacks - Infosec Institute](https://www.infosecinstitute.com/resources/hacking/pdf-file-format-basic-structure/)
