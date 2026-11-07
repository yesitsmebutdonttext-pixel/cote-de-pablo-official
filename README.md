/**
 * Represents the configuration options for a network request client.
 */
interface HttpClientConfig {
  baseUrl: string;
  timeoutMs?: number;
  defaultHeaders?: Record<string, string>;
}

/**
 * Custom error class for API-related failures to keep errors predictable.
 */
class ApiError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
    this.name = 'ApiError';
  }
}

/**
 * A robust, professional HTTP Client wrapper demonstrating Clean Code principles.
 */
class HttpClient {
  private readonly baseUrl: string;
  private readonly timeoutMs: number;
  private readonly defaultHeaders: Record<string, string>;

  constructor(config: HttpClientConfig) {
    if (!config.baseUrl) {
      throw new Error('HttpClient initialization failed: baseUrl is required.');
    }
    this.baseUrl = config.baseUrl;
    this.timeoutMs = config.timeoutMs ?? 5000; // Default to 5 seconds
    this.defaultHeaders = config.defaultHeaders ?? {};
  }

  /**
   * Performs a typed asynchronous GET request.
   * 
   * @template T The expected response data type.
   * @param endpoint The relative URI path.
   * @returns A promise resolving to the parsed response data.
   */
  public async get<T>(endpoint: string): Promise<T> {
    const url = `${this.baseUrl}${endpoint}`;
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.timeoutMs);

    try {
      const response = await fetch(url, {
        method: 'GET',
        headers: {
          'Content-Type': 'application/json',
          ...this.defaultHeaders,
        },
        signal: controller.signal,
      });

      if (!response.ok) {
        throw new ApiError(
          response.status,
          `Request failed with status ${response.status}: ${response.statusText}`
        );
      }

      return (await response.json()) as T;
    } catch (error: unknown) {
      if (error instanceof DOMException && error.name === 'AbortError') {
        throw new Error(`Request timeout exceeded for ${url} after ${this.timeoutMs}ms`);
      }
      // Re-throw custom API errors or wrap unexpected runtime exceptions
      throw error;
    } finally {
      clearTimeout(timeoutId);
    }
  }
}

// --- Usage Example ---
interface UserProfile {
  id: string;
  email: string;
  isActive: boolean;
}

async function fetchUserData(userId: string): Promise<UserProfile | null> {
  const client = new HttpClient({ baseUrl: 'https://api.example.com' });

  try {
    return await client.get<UserProfile>(`/users/${userId}`);
  } catch (error) {
    console.error('Failed to retrieve user profile:', error);
    return null; // Graceful degradation
  }
}
