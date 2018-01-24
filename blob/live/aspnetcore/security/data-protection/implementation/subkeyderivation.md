---
title: "子项派生和经过身份验证的加密"
author: rick-anderson
description: "本文档说明 ASP.NET 核心数据保护的实现详细信息子项派生和身份验证加密。"
ms.author: riande
manager: wpickett
ms.date: 10/14/2016
ms.topic: article
ms.technology: aspnet
ms.prod: asp.net-core
uid: security/data-protection/implementation/subkeyderivation
ms.openlocfilehash: 3927678b7b67b0e521a961e363200bdfe0bdaeb3
ms.sourcegitcommit: 3e303620a125325bb9abd4b2d315c106fb8c47fd
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 01/19/2018
---
# <a name="subkey-derivation-and-authenticated-encryption"></a><span data-ttu-id="019b2-103">子项派生和经过身份验证的加密</span><span class="sxs-lookup"><span data-stu-id="019b2-103">Subkey Derivation and Authenticated Encryption</span></span>

<a name="data-protection-implementation-subkey-derivation"></a>

<span data-ttu-id="019b2-104">密钥链中的大多数键将包含某种形式的平均信息量，并且将有算法信息指出"CBC 模式下加密 + HMAC 验证"或"GCM 加密 + 验证"。</span><span class="sxs-lookup"><span data-stu-id="019b2-104">Most keys in the key ring will contain some form of entropy and will have algorithmic information stating "CBC-mode encryption + HMAC validation" or "GCM encryption + validation".</span></span> <span data-ttu-id="019b2-105">在这些情况下，我们将嵌入的平均信息量称为此密钥的主密钥材料 （或公里） 和我们执行派生将使用为实际的加密操作的密钥的密钥派生函数。</span><span class="sxs-lookup"><span data-stu-id="019b2-105">In these cases, we refer to the embedded entropy as the master keying material (or KM) for this key, and we perform a key derivation function to derive the keys that will be used for the actual cryptographic operations.</span></span>

> [!NOTE]
> <span data-ttu-id="019b2-106">键是抽象的并自定义实现可能不会按如下所示的那样。</span><span class="sxs-lookup"><span data-stu-id="019b2-106">Keys are abstract, and a custom implementation might not behave as below.</span></span> <span data-ttu-id="019b2-107">如果密钥提供其自己的实现`IAuthenticatedEncryptor`而不是使用我们的内置工厂之一，本节中所述的机制不再适用。</span><span class="sxs-lookup"><span data-stu-id="019b2-107">If the key provides its own implementation of `IAuthenticatedEncryptor` rather than using one of our built-in factories, the mechanism described in this section no longer applies.</span></span>

<a name="data-protection-implementation-subkey-derivation-aad"></a>

## <a name="additional-authenticated-data-and-subkey-derivation"></a><span data-ttu-id="019b2-108">其他经过身份验证的数据和子项派生</span><span class="sxs-lookup"><span data-stu-id="019b2-108">Additional authenticated data and subkey derivation</span></span>

<span data-ttu-id="019b2-109">`IAuthenticatedEncryptor`接口用作所有经过身份验证的加密操作的核心接口。</span><span class="sxs-lookup"><span data-stu-id="019b2-109">The `IAuthenticatedEncryptor` interface serves as the core interface for all authenticated encryption operations.</span></span> <span data-ttu-id="019b2-110">其`Encrypt`方法采用两个缓冲区： 纯文本和 additionalAuthenticatedData (AAD)。</span><span class="sxs-lookup"><span data-stu-id="019b2-110">Its `Encrypt` method takes two buffers: plaintext and additionalAuthenticatedData (AAD).</span></span> <span data-ttu-id="019b2-111">纯文本内容流保持不变调用`IDataProtector.Protect`，但 AAD 由系统生成和包括三个组件：</span><span class="sxs-lookup"><span data-stu-id="019b2-111">The plaintext contents flow unchanged the call to `IDataProtector.Protect`, but the AAD is generated by the system and consists of three components:</span></span>

1. <span data-ttu-id="019b2-112">32 位神奇标头 09 F0 C9 F0 标识此版本的数据保护系统。</span><span class="sxs-lookup"><span data-stu-id="019b2-112">The 32-bit magic header 09 F0 C9 F0 that identifies this version of the data protection system.</span></span>

2. <span data-ttu-id="019b2-113">128 位密钥 id。</span><span class="sxs-lookup"><span data-stu-id="019b2-113">The 128-bit key id.</span></span>

3. <span data-ttu-id="019b2-114">从创建目的链形成的可变长度字符串`IDataProtector`，正在执行此操作。</span><span class="sxs-lookup"><span data-stu-id="019b2-114">A variable-length string formed from the purpose chain that created the `IDataProtector` that is performing this operation.</span></span>

<span data-ttu-id="019b2-115">由于 AAD 是为所有三个组件的元组唯一的我们可以使用它从密钥主机派生新密钥，而不是使用密钥主机本身中所有的我们的加密操作。</span><span class="sxs-lookup"><span data-stu-id="019b2-115">Because the AAD is unique for the tuple of all three components, we can use it to derive new keys from KM instead of using KM itself in all of our cryptographic operations.</span></span> <span data-ttu-id="019b2-116">每次调用`IAuthenticatedEncryptor.Encrypt`，发生以下密钥派生过程：</span><span class="sxs-lookup"><span data-stu-id="019b2-116">For every call to `IAuthenticatedEncryptor.Encrypt`, the following key derivation process takes place:</span></span>

<span data-ttu-id="019b2-117">( K_E, K_H ) = SP800_108_CTR_HMACSHA512(K_M, AAD, contextHeader || keyModifier)</span><span class="sxs-lookup"><span data-stu-id="019b2-117">( K_E, K_H ) = SP800_108_CTR_HMACSHA512(K_M, AAD, contextHeader || keyModifier)</span></span>

<span data-ttu-id="019b2-118">在这里，我们正在呼叫 NIST SP800 108 KDF 中计数器模式 (请参阅[NIST SP800-108](http://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-108.pdf)，sec。 5.1) 使用以下参数：</span><span class="sxs-lookup"><span data-stu-id="019b2-118">Here, we're calling the NIST SP800-108 KDF in Counter Mode (see [NIST SP800-108](http://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-108.pdf), Sec. 5.1) with the following parameters:</span></span>

* <span data-ttu-id="019b2-119">密钥派生密钥 (KDK) = K_M</span><span class="sxs-lookup"><span data-stu-id="019b2-119">Key derivation key (KDK) = K_M</span></span>

* <span data-ttu-id="019b2-120">PRF = HMACSHA512</span><span class="sxs-lookup"><span data-stu-id="019b2-120">PRF = HMACSHA512</span></span>

* <span data-ttu-id="019b2-121">标签 = additionalAuthenticatedData</span><span class="sxs-lookup"><span data-stu-id="019b2-121">label = additionalAuthenticatedData</span></span>

* <span data-ttu-id="019b2-122">上下文 = contextHeader | |keyModifier</span><span class="sxs-lookup"><span data-stu-id="019b2-122">context = contextHeader || keyModifier</span></span>

<span data-ttu-id="019b2-123">上下文标头的可变长度和本质就是我们要为其派生 K_E 和 K_H 的算法的指纹。</span><span class="sxs-lookup"><span data-stu-id="019b2-123">The context header is of variable length and essentially serves as a thumbprint of the algorithms for which we're deriving K_E and K_H.</span></span> <span data-ttu-id="019b2-124">密钥修饰符是每次调用随机生成一个 128 位字符串`Encrypt`并提供以确保与绝大多数概率 KE 和 KH 是唯一对于此特定身份验证加密操作，即使所有其他输入给 KDF 为常数。</span><span class="sxs-lookup"><span data-stu-id="019b2-124">The key modifier is a 128-bit string randomly generated for each call to `Encrypt` and serves to ensure with overwhelming probability that KE and KH are unique for this specific authentication encryption operation, even if all other input to the KDF is constant.</span></span>

<span data-ttu-id="019b2-125">CBC 模式下加密 + HMAC 验证操作，|K_E |的密钥长度是对称的块密码，和 |K_H |是的 HMAC 例程的摘要大小。</span><span class="sxs-lookup"><span data-stu-id="019b2-125">For CBC-mode encryption + HMAC validation operations, | K_E | is the length of the symmetric block cipher key, and | K_H | is the digest size of the HMAC routine.</span></span> <span data-ttu-id="019b2-126">对 GCM 加密 + 验证操作 |K_H |= 0。</span><span class="sxs-lookup"><span data-stu-id="019b2-126">For GCM encryption + validation operations, | K_H | = 0.</span></span>

## <a name="cbc-mode-encryption--hmac-validation"></a><span data-ttu-id="019b2-127">CBC 模式下加密 + HMAC 验证</span><span class="sxs-lookup"><span data-stu-id="019b2-127">CBC-mode encryption + HMAC validation</span></span>

<span data-ttu-id="019b2-128">一旦 K_E 生成通过上述机制中，我们生成的随机初始化向量，并运行该对称的块密码算法来加密纯文本。</span><span class="sxs-lookup"><span data-stu-id="019b2-128">Once K_E is generated via the above mechanism, we generate a random initialization vector and run the symmetric block cipher algorithm to encipher the plaintext.</span></span> <span data-ttu-id="019b2-129">然后通过使用密钥 K_H 以生成 MAC 初始化 HMAC 例程将的初始化向量和已加密文本运行</span><span class="sxs-lookup"><span data-stu-id="019b2-129">The initialization vector and ciphertext are then run through the HMAC routine initialized with the key K_H to produce the MAC.</span></span> <span data-ttu-id="019b2-130">下面以图形方式表示此过程和返回值。</span><span class="sxs-lookup"><span data-stu-id="019b2-130">This process and the return value is represented graphically below.</span></span>

![CBC 模式过程并返回](subkeyderivation/_static/cbcprocess.png)

<span data-ttu-id="019b2-132">*输出: = keyModifier | |iv | |E_cbc （K_E，iv，数据） | |HMAC (K_H、 iv | |E_cbc （K_E，iv，数据）)*</span><span class="sxs-lookup"><span data-stu-id="019b2-132">*output:= keyModifier || iv || E_cbc (K_E,iv,data) || HMAC(K_H, iv || E_cbc (K_E,iv,data))*</span></span>

> [!NOTE]
> <span data-ttu-id="019b2-133">`IDataProtector.Protect`实现将[前面预置的幻标头和密钥 id](authenticated-encryption-details.md)到之前将其返回到调用方的输出。</span><span class="sxs-lookup"><span data-stu-id="019b2-133">The `IDataProtector.Protect` implementation will [prepend the magic header and key id](authenticated-encryption-details.md) to output before returning it to the caller.</span></span> <span data-ttu-id="019b2-134">因为神奇的标头和密钥 id 是隐式的一部分[AAD](xref:security/data-protection/implementation/subkeyderivation#data-protection-implementation-subkey-derivation-aad)，因为密钥修饰符作为输入提供给 KDF，这意味着 MAC 进行身份验证的最终返回负载的每个单字节</span><span class="sxs-lookup"><span data-stu-id="019b2-134">Because the magic header and key id are implicitly part of [AAD](xref:security/data-protection/implementation/subkeyderivation#data-protection-implementation-subkey-derivation-aad), and because the key modifier is fed as input to the KDF, this means that every single byte of the final returned payload is authenticated by the MAC.</span></span>

## <a name="galoiscounter-mode-encryption--validation"></a><span data-ttu-id="019b2-135">Galois/计数器模式加密 + 验证</span><span class="sxs-lookup"><span data-stu-id="019b2-135">Galois/Counter Mode encryption + validation</span></span>

<span data-ttu-id="019b2-136">一旦 K_E 生成通过上述机制中，我们将生成随机的 96 位 nonce 和运行对称块加密算法，来加密纯文本，并生成的 128 位身份验证标记。</span><span class="sxs-lookup"><span data-stu-id="019b2-136">Once K_E is generated via the above mechanism, we generate a random 96-bit nonce and run the symmetric block cipher algorithm to encipher the plaintext and produce the 128-bit authentication tag.</span></span>

![GCM 模式过程并返回](subkeyderivation/_static/galoisprocess.png)

<span data-ttu-id="019b2-138">*输出: = keyModifier | |nonce | |E_gcm （K_E，nonce，数据） | |authTag*</span><span class="sxs-lookup"><span data-stu-id="019b2-138">*output := keyModifier || nonce || E_gcm (K_E,nonce,data) || authTag*</span></span>

> [!NOTE]
> <span data-ttu-id="019b2-139">即使 GCM 本机支持 AAD 的概念，我们有一个要仍输入 AAD 仅对原始 KDF 中，选择要传递到 GCM 其 AAD 参数为空字符串。</span><span class="sxs-lookup"><span data-stu-id="019b2-139">Even though GCM natively supports the concept of AAD, we're still feeding AAD only to the original KDF, opting to pass an empty string into GCM for its AAD parameter.</span></span> <span data-ttu-id="019b2-140">这样做的原因是两个折叠。</span><span class="sxs-lookup"><span data-stu-id="019b2-140">The reason for this is two-fold.</span></span> <span data-ttu-id="019b2-141">首先，[以支持灵活性](context-headers.md#data-protection-implementation-context-headers)我们永远不希望将 K_M 直接用作加密密钥。</span><span class="sxs-lookup"><span data-stu-id="019b2-141">First, [to support agility](context-headers.md#data-protection-implementation-context-headers) we never want to use K_M directly as the encryption key.</span></span> <span data-ttu-id="019b2-142">此外，GCM 有一定非常严格的唯一性要求其输入。</span><span class="sxs-lookup"><span data-stu-id="019b2-142">Additionally, GCM imposes very strict uniqueness requirements on its inputs.</span></span> <span data-ttu-id="019b2-143">GCM 加密例程是曾调用对两个或更清晰的概率具有相同 （键，nonce） 的输入数据集对不能超过 2 ^32。</span><span class="sxs-lookup"><span data-stu-id="019b2-143">The probability that the GCM encryption routine is ever invoked on two or more distinct sets of input data with the same (key, nonce) pair must not exceed 2^32.</span></span> <span data-ttu-id="019b2-144">如果我们修复 K_E 我们无法执行多个 2 ^32 加密操作之前运行与我们的 2 ^-32 限制。</span><span class="sxs-lookup"><span data-stu-id="019b2-144">If we fix K_E we cannot perform more than 2^32 encryption operations before we run afoul of the 2^-32 limit.</span></span> <span data-ttu-id="019b2-145">这可能看起来非常大量的操作，但在几天内，这些密钥的正常生存期内良好的高流量 web 服务器可以通过 40 亿个请求。</span><span class="sxs-lookup"><span data-stu-id="019b2-145">This might seem like a very large number of operations, but a high-traffic web server can go through 4 billion requests in mere days, well within the normal lifetime for these keys.</span></span> <span data-ttu-id="019b2-146">保持合规的 2 ^ 我们继续使用 128 位密钥修饰符和 96 位 nonce，因而可真正扩展任何给定 K_M 的可用操作计数-32 概率限制。</span><span class="sxs-lookup"><span data-stu-id="019b2-146">To stay compliant of the 2^-32 probability limit, we continue to use a 128-bit key modifier and 96-bit nonce, which radically extends the usable operation count for any given K_M.</span></span> <span data-ttu-id="019b2-147">为简单起见的设计中，我们将共享 CBC 和 GCM 操作之间的 KDF 代码路径，并且由于 AAD 已被视为 KDF 中没有无需将其转发给 GCM 例程。</span><span class="sxs-lookup"><span data-stu-id="019b2-147">For simplicity of design we share the KDF code path between CBC and GCM operations, and since AAD is already considered in the KDF there is no need to forward it to the GCM routine.</span></span>