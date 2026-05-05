---
title: "Accelerate CPU-based AI inference workloads using Intel AMX on Amazon EC2"
url: "https://aws.amazon.com/blogs/compute/accelerate-cpu-based-ai-inference-workloads-using-intel-amx-on-amazon-ec2/"
date: "Mon, 30 Mar 2026 16:43:10 +0000"
author: "Santosh Kumar"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
<p>This post shows you how to accelerate your AI inference workloads by up to 76% using <a href="https://www.intel.com/content/www/us/en/products/docs/accelerator-engines/what-is-intel-amx.html" rel="noopener noreferrer" target="_blank">Intel Advanced Matrix Extensions (AMX)</a> – an accelerator that uses specialized hardware and instructions to perform matrix operations directly on processor cores – on <a href="https://aws.amazon.com/pm/ec2/" rel="noopener noreferrer" target="_blank">Amazon Elastic Compute Cloud (Amazon EC2)</a> 8th generation instances. You’ll learn when CPU-based inference is cost-effective, how to enable AMX with minimal code changes, and which configurations deliver optimal performance for your models.</p> 
<p>Many organizations find that CPU-based inference is more suitable for their production Artificial Intelligence/Machine Learning (AI/ML) workloads after evaluating factors like cost, operational complexity, and infrastructure compatibility. As more organizations deploy AI solutions, improving how models run on standard CPUs has become a critical cost control strategy for workloads where CPU inference provides the right balance of performance and economics.</p> 
<p><a href="https://my.idc.com/getdoc.jsp?containerId=prUS52530724" rel="noopener noreferrer" target="_blank">IDC</a>, a global market intelligence and advisory firm, projects that worldwide AI spending will reach $632 billion by 2028, growing at a 29% compound annual growth rate from 2024, with inference costs representing a significant portion of operational expenses. <a href="https://www.deloitte.com/us/en/about/press-room/deloitte-2026-tmt-predictions.html" rel="noopener noreferrer" target="_blank">Deloitte</a>, a leading professional services firm specializing in technology consulting and research, forecasts that inference – the running of AI models – will make up two-thirds of all AI compute by 2026, far exceeding initial training costs. This makes optimizing AI/ML inference on CPU crucial for controlling long-term AI/ML operational expenses.</p> 
<p>At the core of AI inference workloads are matrix multiplication operations – the mathematical foundation of neural networks that drives computational demand. These matrix-heavy operations create a performance bottleneck for CPU-based inference, resulting in suboptimal performance for AI/ML workloads. This creates three key challenges for organizations: balancing cost optimization with performance requirements, meeting real-time latency demands, and scaling efficiently with variable workload demands. Intel’s Advanced Matrix Extensions (AMX) technology addresses these challenges by accelerating matrix operations directly on CPU cores, making CPU-based inference competitive and cost-effective.</p> 
<h3>AMX capabilities and architecture</h3> 
<p>AMX supports multiple data formats including <a href="https://www.intel.com/content/www/us/en/content-details/671279/bfloat16-hardware-numerics-definition.html" rel="noopener noreferrer" target="_blank">BF16</a> which preserves the range of 32-bit floating point operations in half the space, INT8 maximizes throughput when accuracy can be slightly compromised, and FP16 offers a balance between the two. This flexibility lets you match precision to your specific needs.</p> 
<p>Introduced in 2023 with 4th Generation <a href="https://www.intel.com/content/www/us/en/products/details/processors/xeon/scalable.html" rel="noopener noreferrer" target="_blank">Intel Xeon Scalable processors</a>, AMX consists of eight 1KB tile registers (specialized on-chip memory for matrix data) and a Tile Matrix Multiply Unit (TMUL – dedicated hardware for matrix calculations) that enables processors to perform 2048 INT8 operations or 1024 BF16 operations per cycle. These tile registers provide efficient matrix storage, reducing memory access overhead and improving computational efficiency for matrix operations central to neural networks.&nbsp;For real-world customer workloads, this translates to significantly faster inference times for <a href="https://aws.amazon.com/what-is/transformers-in-artificial-intelligence/" rel="noopener noreferrer" target="_blank">transformer</a> models, recommendation systems, and natural language processing tasks, while reducing the total cost of ownership through improved resource utilization and lower infrastructure requirements.</p> 
<div class="wp-caption aligncenter" id="attachment_25812" style="width: 567px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/1-ComputeBlog-2473-AMX-Architecture.png"><img alt="Architecture diagram of Intel Advanced Matrix Extensions (AMX) showing the key components: Intel Xeon CPU with AMX support, tile architecture with 8 tiles of 1 KiB each as 2D registers, Tile Matrix Multiply Unit (TMUL) with data flow between them, supported data types (BF16, INT8, FP16), and AMX instruction categories (Configuration, Data Management, Operations)" class=" wp-image-25812" height="453" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/20/1-ComputeBlog-2473-AMX-Architecture.png" width="557" /></a>
 <p class="wp-caption-text" id="caption-attachment-25812">Figure 1: AMX Architecture showing AMX tile registers, processing units, and data flow within CPU core</p>
</div> 
<p><strong><em>Note: </em></strong><em>AMX operations, including tile setup and memory-to-tile data movement (which are handled automatically by the system), introduce small overhead that may outweigh benefits for smaller models or single-batch processing where insufficient matrix operations cannot amortize these costs, making batch size optimization critical for performance gains.</em></p> 
<h2>When to choose CPU inference with AMX</h2> 
<p>CPU inference with AMX acceleration benefits workloads including:</p> 
<p><strong>Batch processing and traditional ML</strong>: Content summarization, recommendation systems, and analytical workloads benefit from CPU’s cost efficiency and ability to handle sparse data structures and branching logic.</p> 
<p><strong>Small to medium-sized models: </strong>Models under 7B parameters and batch sizes of 8-16 samples achieve excellent performance through optimized threading, making CPUs ideal for applications like fraud detection and chatbots.</p> 
<p><strong>Variable demand workloads</strong>: E-commerce systems and applications with unpredictable traffic patterns can quickly scale CPU instances up or down based on demand, avoiding the fixed costs of dedicated accelerator hardware that sits idle during low-traffic periods.</p> 
<p><strong>Complex business logic</strong>: Applications like financial risk assessment and content moderation that need to combine ML predictions with business rules and conditional logic work well on CPUs, which handle mixed workloads better than specialized accelerators.</p> 
<h2>Implementation: AMX optimization with PyTorch</h2> 
<p><a href="https://pytorch.org/" rel="noopener noreferrer" target="_blank">PyTorch</a>, a popular open-source machine learning framework, includes built-in Intel optimizations through <a href="https://www.intel.com/content/www/us/en/developer/tools/oneapi/onednn.html" rel="noopener noreferrer" target="_blank">oneDNN</a> (Intel’s Deep Neural Network library) that automatically use AMX when available. Setup requires installing dependencies and configuring environment variables for optimal performance.</p> 
<h3>Install dependencies</h3> 
<pre><code class="lang-bash"># Install transformers and torch
pip install torch transformers</code></pre> 
<h3>Configure environment variables</h3> 
<p>These environment variables tell oneDNN library how to optimize your inference workload for AMX.</p> 
<ol> 
 <li>Enable AMX instruction set (tells oneDNN to use AMX tiles for matrix operations): <pre><code class="lang-bash">export DNNL_MAX_CPU_ISA=AVX512_CORE_AMX</code></pre> </li> 
 <li>Optimize thread affinity (binds threads to CPU cores for better cache performance): <pre><code class="lang-bash">export KMP_AFFINITY=granularity=fine,compact,1,0</code></pre> </li> 
 <li>Use all available CPU cores for parallel processing: <pre><code class="lang-bash">export OMP_NUM_THREADS=$(nproc)</code></pre> </li> 
 <li>Cache compiled kernels (avoids recompilation overhead on subsequent runs): <pre><code class="lang-bash">export ONEDNN_PRIMITIVE_CACHE_CAPACITY=4096</code></pre> </li> 
 <li>Set default precision to BF16 (enables automatic AMX acceleration): <pre><code class="lang-bash">export ONEDNN_DEFAULT_FPMATH_MODE=bf16</code></pre> </li> 
 <li>(Optional) Enable verbose logging to verify AMX activation: <pre><code class="lang-bash">export ONEDNN_VERBOSE=1</code></pre> </li> 
</ol> 
<h3>BF16 optimization example</h3> 
<p>With environment variables configured, implementing BF16 optimization requires minimal to no code changes. The following example demonstrates how PyTorch automatically leverages AMX tile registers for matrix operations when BF16 precision is used.</p> 
<p><strong>Note:</strong> This is a simplified example for demonstration purposes; adapt the code to your specific use case and requirements.</p> 
<pre><code class="lang-python">import torch
from transformers import AutoTokenizer, AutoModelForCausalLM
import time

# Load model and tokenizer from HuggingFace
model_name = "google/gemma-3-1b-it"

model_revision = "dcc83ea841ab6100d6b47a070329e1ba4cf78752"
tokenizer = AutoTokenizer.from_pretrained(
    model_name,
    revision=model_revision
)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    revision=model_revision
)
# Fix tokenizer padding issue for batch processing
if tokenizer.pad_token is None:
    tokenizer.pad_token = tokenizer.eos_token

# Enable BF16 precision for automatic AMX acceleration
model = model.to(dtype=torch.bfloat16)
model.eval()  # Set to inference mode

# Inference function with BF16 autocast
def run_optimized_inference(prompts):
    inputs = tokenizer(prompts, padding=True, 
                      return_tensors="pt")  # Tokenize input
    
    with torch.no_grad():  # Disable gradients for inference
        with torch.amp.autocast('cpu',
                               dtype=torch.bfloat16):  # BF16 autocast
            outputs = model.generate(
                **inputs,
                max_length=100,     # Set maximum sequence length 
                do_sample=False     # Use greedy decoding
            )
    return outputs

# Example usage with performance measurement
prompts = ["What are the benefits of cloud computing?"]
start_time = time.time()
results = run_optimized_inference(prompts)  # Run BF16-optimized inference
elapsed_time = time.time() - start_time
tokens_generated = len(results[0]) - len(tokenizer.encode(
    prompts[0]))  # Count new tokens

# Display results and performance metrics
print(tokenizer.decode(results[0], skip_special_tokens=True))
print(f"Latency: {elapsed_time*1000:.1f}ms, "
      f"Throughput: {tokens_generated/elapsed_time:.1f} "
      f"tokens/sec")</code></pre> 
<h2>Performance benchmarks</h2> 
<p>To validate AMX performance benefits, we conducted benchmarks across multiple popular language models representing different use cases and model sizes.</p> 
<h3>Benchmarking methodology and environment</h3> 
<p>We tested two improvements: hardware generation advances (m8i vs m7i) and AMX optimization impact (FP32 vs BF16). This shows you both upgrade paths for your workloads.</p> 
<ul> 
 <li><strong>Models tested</strong>: BigBird-RoBERTa-large (355M), Microsoft DialoGPT-large (762M), Google Gemma-3-1b-it (1B), DeepSeek-R1-Distill-Qwen-1.5B (1.5B), Llama-3.2-3B-Instruct (3B), YOLOv5&nbsp;(tested with 30 images at ~1200×800 resolution with 5 iterations for each image)</li> 
 <li><strong>Amazon EC2 instance types</strong>: <a href="https://aws.amazon.com/ec2/instance-types/m8i/" rel="noopener noreferrer" target="_blank">m8i.4xlarge</a>, <a href="https://aws.amazon.com/ec2/instance-types/m7i/" rel="noopener noreferrer" target="_blank">m7i.4xlarge</a> (8<sup>th</sup> &amp; 7<sup>th</sup> gen general-purpose Amazon EC2 instances with 16 vCPUs and 64 GiB memory, both AMX-capable)</li> 
 <li><strong>Batch sizes</strong>: 1, 8, 32&nbsp;(number of input samples processed simultaneously in a single inference call)</li> 
 <li><strong>Iterations</strong>: 5 runs per configuration</li> 
 <li><strong>Comparison types</strong>: 
  <ul> 
   <li>Instance generation comparison (m8i vs m7i performance)</li> 
   <li>AMX optimization impact (32-bit floating-point (FP32) vs Brain Floating Point 16 (BF16) on same instance)</li> 
  </ul> </li> 
 <li><strong>Optimizations</strong>: FP32 baseline vs BF16 AMX</li> 
 <li><strong>Framework</strong>:&nbsp;PyTorch 2.8.0 (which has built-in Intel optimizations)</li> 
 <li><strong>Region</strong>: AWS us-west-2</li> 
 <li><strong>Measurement methodology</strong>: In our benchmarks, ‘inference latency’ represents the complete model inference execution time including input tokenization and full sequence generation (for generative models) or complete forward pass (for non-generative models). Each measurement is the average of 5 iterations after warm-up iterations, excluding model loading time. We use this metric because AMX’s matrix multiplication acceleration improves performance throughout the complete forward pass.</li> 
</ul> 
<p><strong>Note:</strong> Throughout this blog, FP32 refers to the default 32-bit floating-point precision, while BF16 refers to Brain Floating Point 16-bit precision with AMX acceleration enabled.</p> 
<p><strong>Disclaimer</strong>: Performance results are based on internal testing and may vary depending on specific workloads, configurations, and environments.</p> 
<h3>Detailed result: BigBird-RoBERTa-large</h3> 
<p>This benchmark represents document classification, content summarization, and text analysis workloads typical in batch processing where high throughput is desirable and offline inference scenarios where strict latency requirements are not critical.</p> 
<div class="wp-caption alignnone" id="attachment_25811" style="width: 1441px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/2-ComputeBlog-2473-latency-datatypeVsbatch-roberta.png"><img alt="Bar chart comparing BigBird-RoBERTa-large inference latency between m7i and m8i instances with FP32 and BF16 precision across batch sizes 1, 8, and 32, showing 55-67% latency reduction with BF16 AMX." class="wp-image-25811 size-full" height="728" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/20/2-ComputeBlog-2473-latency-datatypeVsbatch-roberta.png" width="1431" /></a>
 <p class="wp-caption-text" id="caption-attachment-25811">Figure 2: m7i.4xlarge vs m8i.4xlarge inference latency comparison for model BigBird-RoBERTa-large (355M parameters)</p>
</div> 
<div class="wp-caption alignnone" id="attachment_25828" style="width: 2507px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/3-ComputeBlog-2473-throughput-roberta.png"><img alt="Bar chart comparing throughput for the BigBird-RoBERTa-large model between m7i.4xlarge and m8i.4xlarge instances across FP32 and BF16(AMX) data types at batch sizes 1, 8, and 32. m8i.4xlarge achieves 4–25% higher throughput, with the largest gain at FP32 batch size 1 (25%, from 1214.29 to 1512.03 tokens/sec). BF16(AMX) batch size 1 reaches the highest overall throughput at 3391.06 tokens/sec on m8i.4xlarge with a 14 % improvement over m7i.4xlarge. Throughput gains with BF16(AMX) are smaller at larger batch sizes (4–5%), as AMX overhead limits scaling for this smaller model." class="wp-image-25828 size-full" height="1274" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/20/3-ComputeBlog-2473-throughput-roberta.png" width="2497" /></a>
 <p class="wp-caption-text" id="caption-attachment-25828">Figure 3: m7i.4xlarge vs m8i.4xlarge throughput comparison for BigBird-RoBERTa-large model across batch sizes 1, 8, and 32</p>
</div> 
<div class="wp-caption alignnone" id="attachment_25829" style="width: 2122px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/4-ComputeBlog-2473-latency-instancetypeVsbatch-roberta.png"><img alt="Bar chart comparing inference latency for bigbird-roberta-large between FP32 and BF16(AMX) data types on m8i.4xlarge and m7i.4xlarge instances at batch sizes 1, 8, and 32, showing BF16(AMX) reduces latency by 55–69% compared to FP32 across all configurations" class="wp-image-25829 size-full" height="1164" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/4-ComputeBlog-2473-latency-instancetypeVsbatch-roberta.png" width="2112" /></a>
 <p class="wp-caption-text" id="caption-attachment-25829">Figure 4: FP32 vs BF16 inference latency comparison for model BigBird-RoBERTa-large (355M parameters) on m7i.4xlarge and m8i.4xlarge instances across batch sizes</p>
</div> 
<p>BigBird-RoBERTa-large model benchmarking demonstrates three key performance improvements. <strong>Figure 2</strong> shows m8i hardware delivers 4-20% latency reduction across batch sizes compared to m7i for both FP32 and BF16 with AMX, providing immediate benefits without application changes. With AMX and BF16, performance gains decrease at higher batch sizes as AMX overhead exceeds benefits for smaller models like BigBird-RoBERTa-large. <strong>Figure 3</strong> validates these improvements with corresponding 4-25% throughput gains, enabling better resource utilization for production applications. <strong>Figure 4</strong> demonstrates that enabling AMX with BF16 optimization provides the most significant impact, reducing m8i latency by 55-67% compared to non-AMX FP32 baseline, enabling 2-3x higher processing capacity and reduced compute costs.</p> 
<p>The analysis above demonstrates the methodology for interpreting benchmark results using BigBird-RoBERTa-large as a representative example. The remaining models (DialoGPT-large, Gemma-3-1b-it, DeepSeek-R1-Distill-Qwen-1.5B, and Llama-3.2-3B-Instruct) follow identical testing procedures and exhibit similar performance patterns, with variations primarily in the magnitude of improvements based on model size and architecture. The comprehensive analysis of five models and their performance implications are synthesized in the following section.</p> 
<h3>Benchmarking result for additional models</h3> 
<p>To validate AMX’s effectiveness across diverse AI workloads, we benchmarked five additional models representing different use cases and model sizes. Each model follows the same testing methodology described above, with performance patterns showing how AMX benefits vary based on model architecture, parameter count, and batch size.</p> 
<h4>DialoGPT-large (762M) – Conversational AI</h4> 
<p>This benchmark represents conversational AI, chatbots, and real-time dialogue systems where low latency and consistent response times are critical for user experience.</p> 
<div class="wp-caption alignnone" id="attachment_25808" style="width: 1441px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/5-ComputeBlog-2473-latency-datatypeVsbatch-dialogpt.png"><img alt="Bar chart comparing inference latency for the DialoGPT-large model between m7i.4xlarge and m8i.4xlarge instances across FP32 and BF16(AMX) data types at batch sizes 1, 8, and 32, showing m8i.4xlarge achieves 9– 25% latency reduction, with the largest improvement at FP32 batch size 32 (25%)" class="size-full wp-image-25808" height="733" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/5-ComputeBlog-2473-latency-datatypeVsbatch-dialogpt.png" width="1431" /></a>
 <p class="wp-caption-text" id="caption-attachment-25808">Figure 5: m7i.4xlarge vs m8i.4xlarge inference latency comparison for model DialoGPT-large (762M parameters)</p>
</div> 
<div class="wp-caption alignnone" id="attachment_25830" style="width: 2507px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/6-ComputeBlog-2473-throughput-dialogpt.png"><img alt="Bar chart comparing throughput for the DialoGPT-large model between m7i.4xlarge and m8i.4xlarge instances across FP32 and BF16(AMX) data types at batch sizes 1, 8, and 32, showing m8i.4xlarge achieves 10–34% higher throughput, with the largest gain at FP32 batch size 32 (34%) and BF16(AMX) batch size 32 reaching the highest overall throughput at 355.9 tokens/sec" class="size-full wp-image-25830" height="1283" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/6-ComputeBlog-2473-throughput-dialogpt.png" width="2497" /></a>
 <p class="wp-caption-text" id="caption-attachment-25830">Figure 6: m7i.4xlarge vs m8i.4xlarge throughput comparison for DialoGPT-large model across batch sizes 1, 8, and 32</p>
</div> 
<div class="wp-caption alignnone" id="attachment_25831" style="width: 2118px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/7-ComputeBlog-2473-latency-instancetypeVsbatch-dialogpt.png"><img alt="Bar chart comparing inference latency for DialoGPT-large between FP32 and BF16(AMX) data types on m8i.4xlarge and m7i.4xlarge instances at batch sizes 1, 8, and 32, showing BF16(AMX) increases latency at batch size 1 (negative improvement of -44% and -45%) but reduces latency at larger batch sizes, with up to 43% reduction at m7i.4xlarge batch size 32" class="size-full wp-image-25831" height="1164" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/7-ComputeBlog-2473-latency-instancetypeVsbatch-dialogpt.png" width="2108" /></a>
 <p class="wp-caption-text" id="caption-attachment-25831">Figure 7: FP32 vs BF16 inference latency comparison for model DialoGPT-large (762M parameters) on m7i.4xlarge and m8i.4xlarge instances across batch sizes</p>
</div> 
<h4>Gemma-3-1b-it (1B) – General Purpose</h4> 
<p>This benchmark represents general-purpose language understanding tasks, content generation, and smaller model deployments suitable for cost-sensitive applications and variable demand workloads.</p> 
<div class="wp-caption alignnone" id="attachment_25805" style="width: 1441px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/8-ComputeBlog-2473-latency-datatypeVsbatch-gemma.png"><img alt="Bar chart comparing inference latency for the Gemma-3-1b-it model between m7i.4xlarge and m8i.4xlarge instances across FP32 and BF16(AMX) data types at batch sizes 1, 8, and 32, showing m8i.4xlarge achieves 7– 17% latency reduction, with the largest improvement at BF16(AMX) batch size 1 (17%)" class="size-full wp-image-25805" height="730" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/8-ComputeBlog-2473-latency-datatypeVsbatch-gemma.png" width="1431" /></a>
 <p class="wp-caption-text" id="caption-attachment-25805">Figure 8: M7i.4xlarge vs M8i.4xlarge inference latency comparison for model Gemma-3-1b-it (1B parameters)</p>
</div> 
<div class="wp-caption alignnone" id="attachment_25832" style="width: 2507px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/9-ComputeBlog-2473-throughput-gemma-1.png"><img alt="Bar chart comparing throughput for the Gemma-3-1b-it model between m7i.4xlarge and m8i.4xlarge instances across FP32 and BF16(AMX) data types at batch sizes 1, 8, and 32, showing m8i.4xlarge achieves 7–20% higher throughput, with the largest gain at BF16(AMX) batch size 1 (20%) and BF16(AMX) batch size 32 reaching the highest overall throughput at 127.8 tokens/sec" class="size-full wp-image-25832" height="1278" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/9-ComputeBlog-2473-throughput-gemma-1.png" width="2497" /></a>
 <p class="wp-caption-text" id="caption-attachment-25832">Figure 9: m7i.4xlarge vs m8i.4xlarge latency and throughput comparison for Gemma-3-1b-it across model batch sizes 1, 8, and 32</p>
</div> 
<div class="wp-caption alignnone" id="attachment_25833" style="width: 2118px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/10-ComputeBlog-2473-latency-instancetypeVsbatch-gemma-1.png"><img alt="Bar chart comparing inference latency for Gemma-3-1b-it between FP32 and BF16(AMX) data types on m8i.4xlarge and m7i.4xlarge instances at batch sizes 1, 8, and 32, showing BF16(AMX) reduces latency by 24–42% at larger batch sizes but slightly increases latency at m7i.4xlarge batch size 1 (-4%), with the best improvement of 42% on m8i.4xlarge at batch size 8" class="size-full wp-image-25833" height="1164" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/10-ComputeBlog-2473-latency-instancetypeVsbatch-gemma-1.png" width="2108" /></a>
 <p class="wp-caption-text" id="caption-attachment-25833">Figure 10: FP32 vs BF16 inference latency comparison for model Gemma-3-1b-it (1B parameters) on m7i.4xlarge and m8i.4xlarge instances across batch sizes</p>
</div> 
<h4>DeepSeek-R1-Distill-Qwen-1.5B (1.5B) – Reasoning</h4> 
<p>This benchmark represents reasoning and analytical workloads, including complex decision-making systems, financial analysis, and applications requiring sophisticated logic processing.</p> 
<div class="wp-caption alignnone" id="attachment_25802" style="width: 1441px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/11-ComputeBlog-2473-latency-instancetypeVsbatch-deepseek.png"><img alt="Bar chart comparing inference latency for the DeepSeek-R1-Distill-Qwen-1.5B model between m7i.4xlarge and m8i.4xlarge instances across FP32 and BF16(AMX) data types at batch sizes 1, 8, and 32, showing m8i.4xlarge achieves 7–16% latency reduction, with the largest improvements at BF16(AMX) batch sizes 1 and 8 (both 16%)" class="size-full wp-image-25802" height="730" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/11-ComputeBlog-2473-latency-instancetypeVsbatch-deepseek.png" width="1431" /></a>
 <p class="wp-caption-text" id="caption-attachment-25802">Figure 11: m7i.4xlarge vs m8i.4xlarge inference latency comparison for model DeepSeek-R1-Distill-Qwen-1.5B (1.5B parameters)</p>
</div> 
<div class="wp-caption alignnone" id="attachment_25834" style="width: 2507px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/12-ComputeBlog-2473-throughput-deepseek.png"><img alt="Bar chart comparing throughput for the DeepSeek-R1-Distill-Qwen-1.5B model between m7i.4xlarge and m8i.4xlarge instances across FP32 and BF16(AMX) data types at batch sizes 1, 8, and 32, showing m8i.4xlarge achieves 8–19% higher throughput, with the largest gains at BF16(AMX) batch sizes 1 and 8 (both 19%) and BF16(AMX) batch size 32 reaching the highest overall throughput at 415.1 tokens/sec" class="size-full wp-image-25834" height="1278" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/12-ComputeBlog-2473-throughput-deepseek.png" width="2497" /></a>
 <p class="wp-caption-text" id="caption-attachment-25834">Figure 12: m7i.4xlarge vs m8i.4xlarge latency and throughput comparison for DeepSeek-R1-Distill-Qwen-1.5B model across batch sizes 1, 8, and 32</p>
</div> 
<div class="wp-caption alignnone" id="attachment_25835" style="width: 2118px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/13-ComputeBlog-2473-latency-instancetypeVsbatch-deepseek-1.png"><img alt="Bar chart comparing inference latency for DeepSeek-R1-Distill-Qwen-1.5B between FP32 and BF16(AMX) data types on m8i.4xlarge and m7i.4xlarge instances at batch sizes 1, 8, and 32, showing BF16(AMX) reduces latency by 17–68% across all configurations, with the largest improvement of 68% on m8i.4xlarge at batch size 8 and consistently strong reductions of 59–66% at larger batch sizes" class="size-full wp-image-25835" height="1164" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/13-ComputeBlog-2473-latency-instancetypeVsbatch-deepseek-1.png" width="2108" /></a>
 <p class="wp-caption-text" id="caption-attachment-25835">Figure 13: FP32 vs BF16 inference latency comparison for model DeepSeek-R1-Distill-Qwen-1.5B (1.5B parameters) on m7i.4xlarge and m8i.4xlarge instances across batch sizes</p>
</div> 
<h4>Llama-3.2-3B-Instruct (3B) – Large model</h4> 
<p>This benchmark represents larger model deployments for complex instruction-following tasks, advanced content generation, and applications requiring higher model capacity while maintaining cost efficiency.</p> 
<div class="wp-caption alignnone" id="attachment_25799" style="width: 1441px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/14-ComputeBlog-2473-latency-instancetypeVsbatch-llama.png"><img alt="Bar chart comparing inference latency for the Llama-3.2-3B-Instruct model between m7i.4xlarge and m8i.4xlarge instances across FP32 and BF16(AMX) data types at batch sizes 1, 8, and 32, showing m8i.4xlarge achieves 8–15% latency reduction, with the largest improvement at FP32 batch size 8 (15%) and consistent gains of 12–14% with BF16(AMX) at smaller batch sizes" class="size-full wp-image-25799" height="730" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/14-ComputeBlog-2473-latency-instancetypeVsbatch-llama.png" width="1431" /></a>
 <p class="wp-caption-text" id="caption-attachment-25799">Figure 14: m7i.4xlarge vs m8i.4xlarge inference latency comparison for model Llama-3.2-3B-Instruct (3B parameters)</p>
</div> 
<div class="wp-caption alignnone" id="attachment_25836" style="width: 2507px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/15-ComputeBlog-2473-throughput-llama.png"><img alt="Bar chart comparing throughput for the Llama-3.2-3B-Instruct model between m7i.4xlarge and m8i.4xlarge instances across FP32 and BF16(AMX) data types at batch sizes 1, 8, and 32, showing m8i.4xlarge achieves 8– 17% higher throughput, with the largest gains at FP32 batch size 8 and BF16(AMX) batch size 1 (both 17%) and BF16(AMX) batch size 32 reaching the highest overall throughput at 187.3 tokens/sec" class="size-full wp-image-25836" height="1278" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/15-ComputeBlog-2473-throughput-llama.png" width="2497" /></a>
 <p class="wp-caption-text" id="caption-attachment-25836">Figure 15: m7i.4xlarge vs m8i.4xlarge latency and throughput comparison for Llama-3.2-3B-Instruct model across batch sizes 1, 8, and 32</p>
</div> 
<div class="wp-caption alignnone" id="attachment_25837" style="width: 2118px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/16-ComputeBlog-2473-latency-instancetypeVsbatch-llama-1.png"><img alt="Bar chart comparing inference latency for Llama-3.2-3B-Instruct between FP32 and BF16(AMX) data types on m8i.4xlarge and m7i.4xlarge instances at batch sizes 1, 8, and 32, showing BF16(AMX) reduces latency by 24–72% across all configurations, with the largest improvements of 72% on both m8i.4xlarge batch size 8 and m7i.4xlarge batch size 8, and consistently strong reductions of 68–70% at batch size 32" class="size-full wp-image-25837" height="1164" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/08/16-ComputeBlog-2473-latency-instancetypeVsbatch-llama-1.png" width="2108" /></a>
 <p class="wp-caption-text" id="caption-attachment-25837">Figure 16: FP32 vs BF16 inference latency comparison for model Llama-3.2-3B-Instruct (3B parameters) on m7i.4xlarge and m8i.4xlarge instances across batch sizes</p>
</div> 
<h4>Yolov5 – Computer vision model</h4> 
<p>This benchmark represents computer vision workloads including object detection, image classification, and real-time video processing applications where consistent throughput is important for production deployments.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td rowspan="2" style="padding: 10px; border: 1px solid #dddddd;"><strong>Instance type</strong></td> 
   <td colspan="2" style="padding: 10px; border: 1px solid #dddddd;"><strong>Inference latency in Sec </strong>(Processing time per image)</td> 
   <td colspan="2" style="padding: 10px; border: 1px solid #dddddd;"> <p><strong>Throughput</strong></p> <p>(Image processed per sec)</p></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">FP32</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">BF16</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">FP32</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">BF16</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">m8i.4xlarge</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">0.034</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">0.029</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">29.23</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">34.63</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">m7i.4xlarge</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">0.038</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">0.031</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">26.39</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">32.28</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>m8i improvement</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>10.5%</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>6.5%</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>10.8%</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>7.3%</strong></td> 
  </tr> 
 </tbody> 
</table> 
<p><strong>Key insights:</strong> m8i instances deliver 7-11% better performance than m7i across both precision formats. Combining hardware upgrade with AMX optimization, m8i with BF16 delivers up to 24% lower latency and 31% higher throughput compared to m7i with FP32.</p> 
<h2>Benchmark result summary</h2> 
<p>The detailed graphs above demonstrate consistent performance patterns across <strong>tested</strong> models. Key findings:</p> 
<h3>M8i vs M7i instance performance</h3> 
<p>m8i instances deliver 9-14% average and up to 20% better performance than m7i across the tested models through hardware advances: up to 4.6x larger L3 cache, higher base frequencies, up to 2.5x higher <a href="https://en.wikipedia.org/wiki/DDR5_SDRAM" rel="noopener noreferrer" target="_blank">DDR5</a> bandwidth, and enhanced AMX execution with FP16 support.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Model</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Use Case</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>m8i average latency improvement*</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">BigBird-RoBERTa-large (355M)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Document analysis</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">10%</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">DialoGPT-large (762M)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Conversational AI</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">14%</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Gemma-3-1b-it (1B)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">General purpose</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">10%</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">DeepSeek-R1 (1.5B)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Reasoning tasks</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">11%</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Llama-3.2-3B (3B)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Large model deployment</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">12%</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">YOLOv5</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Computer vision</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">9%</td> 
  </tr> 
 </tbody> 
</table> 
<p>* Average across all tested configurations (FP32 and BF16 at batch sizes 1, 8, and 32)</p> 
<h3>AMX acceleration impact (FP32 vs BF16)</h3> 
<p>BF16 precision with AMX delivers 21-72% performance improvements at batch sizes of 8 and above compared to FP32 baseline on the same instance type. These results compare FP32 vs BF16 performance on m8i.4xlarge, with performance gains varying by model size and batch configuration. Larger batch sizes show greater AMX benefits.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td rowspan="2" style="padding: 10px; border: 1px solid #dddddd;">Model</td> 
   <td colspan="3" style="padding: 10px; border: 1px solid #dddddd;">Latency improvement (%)</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Batch 1</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Batch 8</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Batch 32</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">BigBird-RoBERTa-large</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">55</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">67</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">63</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">DialoGPT-large</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">– 44*</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">21</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">30</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Gemma-3-1b-it</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">6</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">42</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">24</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">DeepSeek-R1</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">24</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">68</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">59</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Llama-3.2-3B</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">27</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">72</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">68</td> 
  </tr> 
 </tbody> 
</table> 
<p>* <em>At batch size 1, DialoGPT-large’s autoregressive decoding generates tokens sequentially, producing many small matrix operations where AMX tile setup overhead exceeds the acceleration benefit. At batch sizes 8 and above, multiple sequences are processed in parallel, creating larger matrix operations that amortize this overhead and deliver 21-30% improvement.</em></p> 
<h4>Performance patterns by batch size</h4> 
<p>Larger models (1B+ parameters) show consistently better AMX performance across the tested batch sizes:</p> 
<ul> 
 <li><strong>Batch size 1</strong>: Mixed results – larger models show 6-27% improvement, smaller models may experience AMX overhead</li> 
 <li><strong>Batch size 8</strong>: Strong performance gains of 21-72% across the tested models, with larger models showing greater benefits</li> 
 <li><strong>Batch size 32</strong>: Significant improvements of 24-68% for most models, demonstrating AMX’s batch processing strength</li> 
</ul> 
<h4>Batch size optimization guidelines</h4> 
<p>AMX performance scales with batch size, with optimal range varies by model size. Performance saturates beyond batch 16 due to hardware limits including memory bandwidth and compute bottlenecks.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Model Size</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Performance Gain</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Recommended Batch Size</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Notes</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">&lt;1B parameters</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">21-67%</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">8-32</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Batch 1 results vary by architecture*</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1-2B parameters</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">42-68%</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">4-16</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">6-24% gains even at batch 1</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">3B+ parameters</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">27-72%</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1-8</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Benefits across batch sizes</td> 
  </tr> 
 </tbody> 
</table> 
<p>* Encoder models (BigBird) show 55% gains at batch 1; autoregressive models (DialoGPT) may experience overhead.</p> 
<h4>Combined performance benefits</h4> 
<p>When we combine AMX optimization with 8th generation instances (m8i), the performance improvements compound significantly. For example, Llama-3.2-3B-Instruct running with BF16 AMX on m8i instances can achieve up to 76% better performance compared to FP32 inference on m7i instances at optimal batch sizes (batch 8: m7i FP32 45.51s vs m8i BF16 10.93s = 76% improvement; batch 32: m7i FP32 62.60s vs m8i BF16 17.47s = 72% improvement).</p> 
<h3>Throughput scaling</h3> 
<p>Across the tested models, throughput (tokens/sec) increases proportionally with latency reduction. This consistent relationship demonstrates that AMX optimizations translate directly to improved inference efficiency.</p> 
<h3>Price-Performance Analysis: Gemma-3-1b-it Model</h3> 
<p>While m8i.4xlarge instances are priced slightly higher than m7i.4xlarge ($0.847 vs $0.806 per hour in us-west-2), they deliver superior price-performance. To illustrate the economic benefits, we analyzed cost per 1 million tokens using Gemma-3-1b-it as a representative example. M8i delivers up to 13% better price-performance over m7i through hardware generation advances, with both instances running BF16 AMX.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td rowspan="2" style="padding: 10px; border: 1px solid #dddddd;"><strong>Batch Size</strong></td> 
   <td rowspan="2" style="padding: 10px; border: 1px solid #dddddd;"><strong>Data Type</strong></td> 
   <td colspan="2" style="padding: 10px; border: 1px solid #dddddd;"><strong>m7i.4xlarge</strong></td> 
   <td colspan="2" style="padding: 10px; border: 1px solid #dddddd;"><strong>m8i.4xlarge</strong></td> 
   <td rowspan="2" style="padding: 10px; border: 1px solid #dddddd;"><strong>Price-Performance improvement</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Throughput<br /> </strong>(tokens/sec)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>$ per 1M token</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Throughput<br /> </strong>(tokens/sec)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>$ per 1M token</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">BF16(AMX)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">14.3</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$15.66</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">17.2</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$13.67</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">13%</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">8</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">BF16(AMX)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">71</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$3.16</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">82.3</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$2.86</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">9%</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">32</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">BF16(AMX)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">119.1</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$1.88</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">127.8</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$1.84</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">2%</td> 
  </tr> 
 </tbody> 
</table> 
<p>Combining the hardware upgrade with BF16 AMX optimization delivers up to 44% better price-performance compared to FP32 on m7i.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td rowspan="2" style="padding: 10px; border: 1px solid #dddddd;"><strong>Batch Size</strong></td> 
   <td colspan="3" style="padding: 10px; border: 1px solid #dddddd;"><strong>m8i.4xlarge</strong></td> 
   <td colspan="3" style="padding: 10px; border: 1px solid #dddddd;"><strong>m7i.4xlarge</strong></td> 
   <td rowspan="2" style="padding: 10px; border: 1px solid #dddddd;"> <p><strong>&nbsp;</strong></p> <p><strong>Price-Performance improvement</strong></p></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Data Type</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Throughput<br /> </strong>(tokens/sec)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>$ per 1M token</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Data Type</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Throughput<br /> </strong>(tokens/sec)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>$ per 1M token</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">BF16(AMX)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">17.2</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$13.67</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">FP32</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">14.9</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$15.03</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">9%</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">8</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">BF16(AMX)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">82.3</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$2.86</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">FP32</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">44.1</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$5.08</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">44%</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">32</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">BF16(AMX)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">127.8</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$1.84</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">FP32</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">89.2</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$2.51</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">27%</td> 
  </tr> 
 </tbody> 
</table> 
<h4>Key findings from the price-performance analysis:</h4> 
<ul> 
 <li><strong>Combined optimization delivers up to 44% better price-performance</strong>: m8i with AMX and BF16 outperforms m7i with FP32 at batch size 8 – consistent with our batch size optimization guidelines where batch sizes of 4-16 deliver optimal results for 1B models like Gemma-3-1b-it, achieving $2.86 per 1M tokens for applications like chatbots and fraud detection.</li> 
 <li><strong>Larger batches maximize cost efficiency</strong>: Batch size 32 reduces costs further to $1.84 per 1M tokens, a 27% improvement over m7i FP32 – ideal for throughput-oriented workloads like content summarization and recommendation systems where latency requirements are flexible.</li> 
</ul> 
<h3>Production deployment recommendation</h3> 
<ul> 
 <li><strong>BF16 AMX</strong>:&nbsp;Delivers 21-72% performance improvements at recommended batch sizes while maintaining model accuracy, making it suitable for production workloads including fraud detection systems, content moderation, and real-time recommendation engines</li> 
 <li><strong>Batch processing</strong>: Target batch sizes of 4-16 based on your use case – smaller batches (1-4) for latency-sensitive applications like chatbots, larger batches (8-16) for throughput-focused scenarios like document analysis and offline processing</li> 
 <li><strong>Instance selection</strong>:&nbsp;m8i instances provide consistent 9-14% performance improvements over m7i, delivering immediate ROI for existing CPU inference workloads without requiring application changes</li> 
 <li><strong>Model size consideration</strong>:&nbsp;Larger models (1B+ parameters) show better AMX utilization across batch sizes, making them ideal candidates for m8i deployment in complex reasoning and content generation applications</li> 
</ul> 
<h2>Conclusion and next steps</h2> 
<p>By using Intel AMX on Amazon EC2 8th generation instances, you can achieve substantial performance improvements for AI inference workloads. Our benchmarks demonstrate&nbsp;up to 72% performance improvements across popular language models, making CPU inference more competitive for batch processing, real-time applications, recommender systems, and variable demand workloads while delivering substantial cost savings through improved resource utilization.</p> 
<p>Key takeaways<strong>:</strong></p> 
<ul> 
 <li><strong>BF16 AMX optimization</strong>&nbsp;delivers up to 72% performance improvements across model sizes, with batch 8 showing 21-72% gains and batch 32 showing 24-68% gains</li> 
 <li><strong>Batch sizes of 4-8 </strong>provide optimal performance for most models—DialoGPT achieves 21% improvement in latency at batch 8, while Llama-3.2-3B achieves 72% improvement</li> 
 <li><strong>8th generation instances</strong>&nbsp;deliver up to 14% performance improvements over m7i across the tested workloads</li> 
 <li><strong>Combined optimizations</strong>&nbsp;(m8i + BF16 AMX) can achieve compound performance improvements up to 76% in optimal configurations (vs m7i FP32), making CPU inference highly competitive for cost-sensitive applications</li> 
 <li><strong>M8i instances deliver up to 13% better price-performance vs m7i</strong> (lower cost per 1M tokens), based on our analysis of the Gemma-3-1b-it model</li> 
 <li><strong>Proper environment configuration</strong> is critical for AMX activation</li> 
</ul> 
<p><strong>You can implement these optimizations immediately. </strong>AMX hardware acceleration combined with PyTorch’s Intel-specific enhancements requires configuring environment variables while delivering substantial speed gains. Begin with BF16 optimization on your existing models, then explore INT8 quantization for additional gains.</p> 
<h3>Next steps:</h3> 
<ol> 
 <li>Launch an Intel based&nbsp;Amazon EC2 8th generation instance (m8i.4xlarge)</li> 
 <li>Install PyTorch (includes built-in Intel optimizations)</li> 
 <li>Configure AMX environment variables</li> 
 <li>Measure performance improvements</li> 
 <li>Scale your optimized inference workloads</li> 
</ol> 
<h2>Additional resources</h2> 
<ul> 
 <li><a href="https://www.intel.com/content/www/us/en/products/docs/accelerator-engines/what-is-intel-amx.html" rel="noopener noreferrer" target="_blank">Intel AMX documentation</a></li> 
 <li><a href="https://aws.amazon.com/ec2/instance-types/m8i/" rel="noopener noreferrer" target="_blank">Amazon EC2 m8i instances</a></li> 
 <li><a href="https://pytorch.org/tutorials/recipes/recipes/tuning_guide.html" rel="noopener noreferrer" target="_blank">PyTorch Intel optimizations guide</a></li> 
 <li><a href="https://huggingface.co/models" rel="noopener noreferrer" target="_blank">HuggingFace model hub</a></li> 
 <li><a href="https://github.com/oneapi-src/oneDNN" rel="noopener noreferrer" target="_blank">oneDNN library documentation</a></li> 
</ul>
