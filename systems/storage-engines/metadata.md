---
description: Adapted from Ruan Yifeng's blog.
---

# 😇 Metadata

Metadata is data that describes other data. This definition is accurate, but it can feel abstract, so it is easier to start with an example.

The following passage is from Chekhov's short story *The Man in a Case*. It describes a woman named Varenka:

> She was no longer very young, about thirty years old, tall, well-built, with dark eyebrows and rosy cheeks. In short, she was not a young girl but a lively, noisy woman who kept humming Little Russian lyric songs, laughed loudly, and often burst into ringing laughter: ha, ha, ha!

This passage provides several types of information: age, around thirty; height, tall; appearance, well-built with dark eyebrows and rosy cheeks; personality, lively, noisy, often humming Little Russian lyric songs, and laughing loudly. With this information, we can roughly imagine what Varenka is like. By extension, if we provide the same categories of information, we can also infer the appearance and character of other people.

In this example, "age", "height", "appearance", and "personality" are metadata, because they are data or information used to describe specific data or information.

Of course, these metadata fields are not precise enough to describe a person's full profile. Most people have filled in a personal information form at some point, including fields such as name, gender, ethnicity, political affiliation, ID photo, education, professional title, and so on. That fuller set of fields is a more complete metadata schema for describing a person.

Metadata is everywhere in daily life. **Whenever there is a class of things, we can define a corresponding set of metadata for that class.**

People who take digital photos know that every digital photo contains `EXIF` information. EXIF is metadata used to describe digital images. According to the `Exif 2.1` standard, it mainly includes the following fields:

> Image Description: the image description or source, usually indicating the tool that generated the image. Artist: the author; some cameras allow the user to enter a name. Make: the manufacturer. Model: the device model. Orientation: the image orientation; some cameras support it and some do not. XResolution/YResolution: the horizontal and vertical resolution. ResolutionUnit: the resolution unit, usually PPI. Software: the software or firmware version. DateTime: date and time. YCbCrPositioning: chroma positioning. ExifOffset: the location where Exif information is written in the file; some software does not display it. ExposureTime: exposure time, namely shutter speed. FNumber: aperture value. ExposureProgram: the programmed auto-exposure setting, which differs across cameras and may include shutter priority or aperture priority. ISO speed ratings: sensitivity. ExifVersion: Exif version. DateTimeOriginal: creation time. DateTimeDigitized: digitization time. ComponentsConfiguration: image component configuration, usually the color-composition scheme. CompressedBitsPerPixel(BPP): bits per pixel after compression, indicating the compression level. ExposureBiasValue: exposure compensation. MaxApertureValue: maximum aperture. MeteringMode: metering mode, such as average metering, center-weighted metering, or spot metering. Lightsource: light source, usually white-balance settings. Flash: whether flash was used. FocalLength: focal length, generally the physical focal length of the lens; some software applies a factor to display the equivalent focal length for a 35 mm camera. MakerNote(User Comment): maker notes, comments, or records. FlashPixVersion: FlashPix version, supported by some models. ColorSpace: color space. ExifImageWidth(Pixel X Dimension): image width in horizontal pixels. ExifImageLength(Pixel Y Dimension): image height in vertical pixels. Interoperability IFD: an interoperability extension pointer related to TIFF files; the specific meaning is unclear. FileSource: source file. Compression: compression ratio.

Here is another example. In the movie database [IMDB](https://www.imdb.com/), each film has its own information page. `IMDB` also defines a metadata schema to describe each film. The following are its top-level metadata categories; each category contains second-level metadata, and together they describe a film from more than one hundred aspects:

> Cast and Crew, Company Credits, Basic Data, Plot & Quotes, Fun Stuff, Links to Other Sites, Box Office and Business, Technical Info, Literature, and Other Data.

The greatest benefit of metadata is that it makes information description and classification structured, which in turn makes machine processing possible.
