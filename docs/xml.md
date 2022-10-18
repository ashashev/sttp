# XML

Adding XML encoding/decoding support is a matter of providing a [body serializer](https://sttp.softwaremill.com/en/latest/requests/body.html) and/or a [response body specification](https://sttp.softwaremill.com/en/latest/responses/body.html) (similarly as it is done for JSON format). The process of adding integrations is fairly easy, and for now, one guide on how to use scalaxb tool is provided.

## scalaxb

If you possess the XML Schema definition file (`.xsd` file) consider using the scalaxb tool, which would generate needed models and serialization/deserialization logic. To use the tool please follow the documentation on [setting up](https://scalaxb.org/setup) and [running](https://scalaxb.org/running-scalaxb) scalaxb.

After code generation, create the `SttpScalaxbApi` trait (or trait with another name of your choosing) and add the following code snippet:

```scala
import generated.defaultScope // import may differ depending on location of generated code
import scalaxb.`package`.{fromXML, toXML} // import may differ depending on location of generated code
import scalaxb.{CanWriteXML, XMLFormat} // import may differ depending on location of generated code
import sttp.client3.{BodySerializer, ResponseAs, ResponseException, StringBody, asString}
import sttp.model.MediaType

import scala.xml.{NodeSeq, XML}

trait SttpScalaxbApi {
  case class XmlElementLabel(label: String)

  implicit def scalaxbBodySerializer[B](implicit format: CanWriteXML[B], label: XmlElementLabel): BodySerializer[B] = { (b: B) =>
    val nodeSeq: NodeSeq = toXML[B](obj = b, elementLabel = label.label, scope = defaultScope)
    StringBody(nodeSeq.toString(), "utf-8", MediaType.ApplicationXml)
  }

  implicit def deserializeXml[B](implicit decoder: XMLFormat[B]): String => Either[Exception, B] = { (s: String) =>
    try {
      Right(fromXML[B](XML.loadString(s)))
    } catch {
      case e: Exception => Left(e)
    }
  }

  def asXml[B: XMLFormat]: ResponseAs[Either[ResponseException[String, Exception], B], Any] =
    asString.mapWithMetadata(ResponseAs.deserializeRightWithError(deserializeXml[B]))
      .showAs("either(as string, as xml)")
}
```
This would add `BodySerializer` needed for serialization and `asXml` method needed for deserialization. Please notice, that `fromXML`, `toXML`, `CanWriteXML`, `XMLFormat` and `defaultScope` are members of code generated by scalaxb.


Next to this trait, you might want to introduce `sttpScalaxb`
package object to simplify imports.
```scala
package object sttpScalaxb extends SttpScalaxbApi
```

From now on, XML serialization/deserialization would work for all classes generated from `.xsd` file as long as `XMLFormat` for the type in the question and `XmlElementLabel` for the top XML node would be implicitly provided in the scope.

Usage example:
```scala
val backend: SttpBackend[Identity, Any] = HttpClientSyncBackend()
val requestPayload = Outer(Inner(42, b = true, "horses"), "cats") // `Outer` and `Inner` classes are generated by scalaxb from xsd file

import sttpScalaxb._ // imports sttp related serialization / deserialization logic
implicit val label = XmlElementLabel("outer") // gives needed XmlElementLabel for the top XML node
import generated.Generated_OuterFormat // imports member of code generated by scalaxb, that provides `XMLFormat` for `Outer` type; this import may differ depending on location of generated code

val response: Identity[Response[Either[ResponseException[String, Exception], Outer]]] =
  basicRequest
    .post(uri"...")
    .body(requestPayload)
    .response(asXml[Outer])
    .send(backend)
```