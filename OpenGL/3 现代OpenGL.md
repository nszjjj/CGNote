## DSA

DSA (Direct State Access) 是从 OpenGL 4.5 版本开始引入的一套 API，它改变了 OpenGL 管理对象状态的方式，旨在提高性能和简化代码。 在 DSA 之前，OpenGL 使用的是一种基于状态机的 API，需要频繁地绑定和解绑对象才能修改其状态。 DSA 使用命名对象 (named objects) 来表示 OpenGL 对象，例如纹理、缓冲区、帧缓冲区等。 这些对象由一个唯一的整数 ID 来标识。 通过 DSA 可以直接使用这些 ID 来操作对象的状态，而无需绑定它们。
